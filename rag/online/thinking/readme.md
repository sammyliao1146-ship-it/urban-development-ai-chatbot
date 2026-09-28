# Thinking RAG

本目录封装由 LangGraph 驱动的 Thinking 模式。它使用单个 Orchestrator 组织输入检查、题型路由、Planner、训练版 RAG、受限纠错、Web/外部工具、冲突处理、Human-in-the-loop、答案证据检查、Checkpoint 和记忆。

Thinking 是“受限 Agentic RAG”，不是多 Agent、Deep Research 或无限反思系统。它不参加 A/B Test，只使用登记并激活的训练版 Chunker、Embedding、Cross-Encoder 和对应索引。训练模型或索引不可用时必须明确失败或进入可恢复流程，不得静默回退到 Semantic Chunking + OpenAI Embedding Baseline。

训练层当前延期时，生产配置必须设置 `THINKING_ENABLED=false`。Graph 路由、Checkpoint 和 Human Interrupt 可以在测试中使用显式 Fake Retriever/Fake Model 验证，但测试替身不得注册为生产模型、不得生成生产 Snapshot，也不得成为在线回退路径。

## 固定检索路径

本地 Retrieval 固定使用同一训练版 Retrieval Snapshot：

```text
训练 Chunk 对应的 bm25s BM25
            +
训练 Embedding 对应的 pgvector KNN/HNSW
            ↓
           RRF
            ↓
去重、来源数量限制、相邻 Chunk 处理
            ↓
训练 Cross-Encoder Reranker
            ↓
统一 Evidence
```

BM25 是训练版 Snapshot 内的确定性词法索引，不属于通用模型回退。BM25、Dense Index、Chunk 映射和 Reranker 必须来自兼容的 Snapshot，版本不匹配时拒绝执行。

## Thinking 主流程

```text
START
  ↓
load_context：加载短期窗口、滚动摘要、相关长期记忆
  ↓
input_guard：检查空输入、实体、时间、地点、单位、歧义和错误前提
  ├─ clear ───────────────────────────────┐
  ├─ auto_rewrite → query_rewrite（最多一次）│
  ├─ incorrect_premise → 保留警告后继续查证   │
  └─ need_user → human_interrupt           │
                                              ↓
question_router：按问题类型选择有界路径
  ├─ simple_fact → local_retrieve
  ├─ ambiguous_context → query_rewrite 或 human_interrupt
  ├─ comparison / causal / multi_hop → planner（最多三个子问题）
  ├─ current / latest → local_retrieve，证据过期或不足时才 web_search
  ├─ structured / numeric → 白名单 SQL/Data API Tool
  └─ out_of_scope → abstain
                                              ↓
orchestrator：执行计划，不创建多个 Agent
  ↓
local_retrieve（最多两轮）/ approved_tool / web_search（最多一次）
  ↓
evidence_grade：检查相关性、覆盖度、可信度、时效性和可回答性
  ├─ sufficient → conflict_resolution
  ├─ retry_local → 有界第二轮本地检索
  ├─ need_web → 一次 Web Search
  ├─ need_user → human_interrupt
  └─ insufficient → abstain
  ↓
conflict_resolution：识别数值、时间、定义、版本和来源冲突
  ├─ resolved / explainable → answer_generate_candidate
  └─ critical_unresolved → human_interrupt
  ↓
answer_generate_candidate：生成完整但不对外发送的最终候选文本
  ↓
bind_citations：把候选答案 Claim 绑定到实际 Evidence
  ↓
final_grounding：检查最终候选文本、Claim、Evidence 和 Citation
  ├─ pass → emit_answer
  ├─ critical_gap → abstain 或 human_interrupt
  └─ internal_error → fail
  ↓
persist_run → optional_memory_write → END
```

`answer_generate_candidate` 产生完整候选文本，但不直接作为 SSE 答案发送。`bind_citations` 和 `final_grounding` 检查的必须是将要返回给用户的同一份文本；检查通过后不得再次调用 LLM 改写。`emit_answer` 只把已经验证并冻结的文本分块发送为 `token` 和 `citation` 事件，避免先流出无依据内容再撤回。代价是 Thinking 的首个答案 Token 会晚于 Grounding，但执行状态仍可实时发送。

## Input Guard 与问题纠错

`input_guard` 只做结构化判断，不直接改变原始用户消息。至少输出：

- `clear`：问题足够明确，可以直接路由；
- `auto_rewrite`：缺失信息可以从当前会话可靠补全；
- `need_user`：主体、地点、时间、指标或目标存在关键歧义，必须询问；
- `incorrect_premise`：问题可能包含错误前提，后续回答必须明确指出，不能顺着错误前提生成；
- `invalid`：空输入、仅噪声或无法形成任务。

自动改写必须同时保存 `original_query` 和 `effective_query`，保留主体、时间、地点、单位以及用户限制。改写文本只是检索输入，不能伪装成用户原话；Query Rewrite 全图最多一次。

## 按题型路由

路由器输出结构化 `question_type`、`route`、`reason_code` 和 `needs_freshness`。`reason_code` 只记录简短、可审计的业务原因，不保存或暴露模型私有思维链。

| 题型 | 默认路径 | 限制 |
| --- | --- | --- |
| 简单事实、定义、单点查询 | 本地训练版 RAG | 不调用 Planner 和 Web |
| 模糊或指代不清 | 一次改写或 Human Interrupt | 不猜测关键条件 |
| 对比、因果、多条件、多跳 | Planner → Orchestrator → 本地 RAG | 最多三个子问题 |
| 最新政策、当前事件、实时数据 | 先本地检索，再按时效性决定 Web | Web 最多一次 |
| 结构化数值或字段查询 | 白名单 SQL/Data API Tool | 参数校验、超时和结果标准化 |
| 证据不足或问题超出语料 | Web、Human Interrupt 或拒答 | 不使用模型常识补写事实 |

Fast 模式不复用该复杂路由器；Fast 继续走固定低延迟 Pipeline。

## 检索纠错与答案纠错

检索纠错采用有界的 Corrective RAG 思路：先评估证据，再决定是否进行第二轮本地检索、一次 Web Search、询问用户或拒答。第二轮本地检索可以扩展检索窗口、调整 metadata filter 或执行 Planner 已生成的子查询，但不得突破训练版 Snapshot，也不得形成无限 Query Rewrite。

答案纠错不是多轮自我反思。`final_grounding` 只对完整最终候选执行一次结构化检查：

- 关键事实、数字、日期、版本和结论是否能映射到实际进入上下文的 Evidence；
- Citation 是否存在、可定位且与 Claim 一致；
- 是否出现 Evidence 未支持的新增事实；
- 冲突是否被如实呈现；
- 证据不足时是否明确拒答。

Grounding 失败后不自动调用 LLM“删除后重写”，因为新文本仍可能引入新 Claim。关键证据缺失时直接拒答或进入 Human Interrupt；如果以后需要自动移除 Claim，只能通过不产生新事实的确定性渲染器，并对渲染结果执行最终一致性断言。

## Web、工具和 Human-in-the-loop

Web Search 只在用户明确要求最新信息、本地证据不足或本地证据时效性不满足问题时触发，最多一次。网页内容必须先转换为统一 Evidence，并作为不可信外部数据处理，不能直接成为系统指令。

SQL/Data API 和未来 MCP 工具统一通过白名单 Tool 接口调用。工具调用由单 Orchestrator 决定，并配置次数、超时、结果大小和错误边界。当前不自动发现并尝试所有工具。

以下情况进入 Human Interrupt：关键歧义无法安全改写、关键数据冲突无法裁决、工具需要用户补充必要参数，或用户必须在多个有效解释之间做选择。恢复执行必须使用 Checkpoint，且不能重复写消息或重复调用已经成功的工具。

## 流式输出边界

LangGraph `astream`/`astream_events` 用于传输安全的执行进度，不代表展示 Chain of Thought。允许发送：

- 当前公开节点状态和简短计划摘要；
- 检索轮次、候选数、来源类型和耗时；
- 工具开始、结束、失败和 Human Interrupt；
- 通过 Grounding 后冻结文本的 Token 与 Citation。

禁止发送隐藏推理、原始 Prompt、完整 Graph State、Checkpoint、未脱敏工具参数、密钥以及未通过检查的候选答案。

## 当前明确不做

- 不实现 FLARE 式逐句或逐 Token“边生成边检索”；
- 不把 `astream` 描述为边生成边检索；
- 不实现 Self-RAG 的反思 Token 训练和多轮自我批判；
- 不实现 RAPTOR 递归聚类、摘要树和层级检索；
- 不实现 GraphRAG、多 Agent、Deep Research 或无限工具探索。

未来如果普通混合检索在长文档全局问题上经评估确认不足，可以在 `rag/pipeline/retrieval/` 下新增 `hierarchical_retriever` 接口研究 RAPTOR；它必须保留摘要节点到原始 Chunk 的引用链，并作为独立版本评估，不能未经测试直接替换当前检索链。

具体 Graph State、Node、条件边、计数器和终止条件定义在 `rag/orchestration/graph/`。内部结构化契约放在 `rag/orchestration/schema/`，Prompt 放在 `rag/orchestration/prompt/`，工具和数据源分别放在 `rag/integration/tool/`、`rag/knowledge/data_source/`。
