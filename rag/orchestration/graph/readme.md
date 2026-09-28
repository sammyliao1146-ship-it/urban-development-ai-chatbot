# Thinking LangGraph

本目录定义 Thinking 模式的 Graph State、Node、条件边、Checkpoint 连接方式和停止条件。这里只负责工作流编排，不实现具体 BM25/KNN/RRF/Rerank 算法，不保存模型文件，也不直接处理 FastAPI Request/Response。

## Graph State 最小字段

未来实现应使用项目内部 Schema，而不是直接把任意字典、LangChain Document 或后端 API Schema 作为全图状态。至少包含：

- 标识：`conversation_id`、`run_id`、`checkpoint_thread_id=run_id`、`graph_version`；
- 问题：`original_query`、`effective_query`、`input_assessment`；
- 路由：`question_type`、`route`、`needs_freshness`、`reason_code`；
- 计划：结构化 `plan`、最多三个 `subqueries`、当前步骤；
- 检索：`retrieval_snapshot_id`、候选、RRF/Rerank 结果和统一 `evidence`；
- 检查：`evidence_grade`、`conflicts`、`grounding_result`；
- 工具：允许的 Tool、调用结果、错误和幂等键；
- 输出：`answer_candidate`、`final_answer`、`citations`、拒答原因和内容哈希；
- 记忆：本次加载的上下文、待审核的记忆候选和允许写入的记忆变更；
- 中断：`interrupt_reason`、用户需要补充的字段和恢复数据；
- 计数器：`rewrite_count`、`local_retrieval_count`、`web_search_count`、`tool_call_count`；
- 控制：超时、取消标记、状态、错误和终止原因。

`original_query` 永远不被覆盖。每个新 Run 必须创建全新的临时 State，不能继承上一 Run 的计数器、Plan、Evidence、Conflict、Candidate 或 Tool Result。多轮上下文只由 `load_context` 从 SQL 消息、滚动摘要和长期记忆读取。所有外部内容必须先标准化为 Evidence 或 Tool Result，不能把网页文本直接拼成控制 Graph 的指令。

## Node 边界

| Node | 职责 | 不负责 |
| --- | --- | --- |
| `load_context` | 加载短期窗口、滚动摘要和相关长期记忆 | 改写问题、生成答案 |
| `input_guard` | 判断清晰度、关键歧义、实体/时间/单位和错误前提 | 直接检索、静默修改用户原话 |
| `query_rewrite` | 根据可靠上下文生成一次 `effective_query` | 无限尝试、改变用户约束 |
| `question_router` | 输出题型、路径和时效性要求 | 执行工具或自由探索 |
| `planner` | 为复杂问题生成最多三个有序子问题 | 生成隐藏长篇推理 |
| `orchestrator` | 按结构化计划调度现有 Node | 创建或模拟多个 Agent |
| `local_retrieve` | 调用训练版 Snapshot 检索接口 | 临时切换 Baseline 或自行构建索引 |
| `approved_tool` | 调用白名单 SQL/Data API/外部工具 | 自动发现并尝试任意工具 |
| `web_search` | 执行最多一次受控 Web Search 并标准化结果 | 直接将网页当 Prompt 指令 |
| `evidence_grade` | 评估相关性、覆盖度、可信度、时效性和可回答性 | 生成最终答案 |
| `conflict_resolution` | 检测并解释数值、日期、定义、版本和来源冲突 | 无证据裁决关键冲突 |
| `answer_generate_candidate` | 基于已选 Evidence 生成完整内部候选文本 | 向 SSE 发送候选 Token |
| `bind_citations` | 将候选文本中的 Claim 绑定到实际 Evidence | 引用未进入上下文的材料 |
| `final_grounding` | 对最终候选文本执行一次 Claim-Evidence-Citation 检查 | 多轮自我反思或检查后再调用 LLM 改写 |
| `emit_answer` | 冻结通过检查的文本并分块发送 Token/Citation | 生成、改写或添加任何事实 |
| `human_interrupt` | 保存 Checkpoint 并请求必要输入 | 长时间占用进程或 Redis 长锁 |
| `persist_run` | 保存用户消息、最终结果、拒答和运行状态 | 自动写入长期记忆 |
| `memory_candidate_write` | 只写入通过长期记忆策略的稳定事实 | 把答案、拒答或模型推测直接当用户事实 |
| `abstain` | 返回证据不足、范围外或所需信息缺失 | 用模型常识补齐事实 |
| `fail` | 返回可追踪错误和恢复信息 | 静默回退通用 RAG |

## 条件边

```text
START → load_context → input_guard

input_guard.clear              → question_router
input_guard.auto_rewrite       → query_rewrite
input_guard.incorrect_premise  → question_router       # 保留前提警告并继续检索查证
input_guard.need_user          → human_interrupt
input_guard.invalid            → abstain
query_rewrite                  → question_router

question_router.simple_fact    → local_retrieve
question_router.ambiguous      → query_rewrite | human_interrupt  # 受 rewrite_count 限制
question_router.complex        → planner
question_router.current        → local_retrieve
question_router.structured     → approved_tool
question_router.need_user      → human_interrupt
question_router.out_of_scope   → abstain
planner                        → orchestrator
orchestrator                   → local_retrieve | approved_tool

local_retrieve                 → evidence_grade
approved_tool                  → evidence_grade
web_search                     → evidence_grade

evidence_grade.sufficient      → conflict_resolution
evidence_grade.retry_local     → local_retrieve       # 仅计数器允许时
evidence_grade.need_web        → web_search           # 仅策略和计数器允许时
evidence_grade.need_user       → human_interrupt
evidence_grade.insufficient    → abstain

conflict_resolution.resolved   → answer_generate_candidate
conflict_resolution.explain    → answer_generate_candidate
conflict_resolution.need_user  → human_interrupt

answer_generate_candidate      → bind_citations
bind_citations                 → final_grounding
final_grounding.pass           → emit_answer
final_grounding.critical_gap   → abstain | human_interrupt
final_grounding.internal_error → fail

emit_answer      → persist_run → memory_candidate_write → END
abstain          → persist_run → END
fail             → END
```

Web Search 结果回到 `evidence_grade`，而不是直接进入 Generation。Human Interrupt 恢复后回到保存的明确继续点，不能从 `START` 无条件重跑整个流程。

## 有界执行规则

- `rewrite_count <= 1`；
- `local_retrieval_count <= 2`；
- `web_search_count <= 1`；
- Planner 的 `subqueries <= 3`；
- `tool_call_count` 使用工具配置的显式上限，默认不能无限调用；
- `final_grounding` 只运行一次，不进入生成—反思—重写循环；检查通过后不再调用 LLM；
- 同时配置 LangGraph `recursion_limit` 和业务计数器，不能只依赖其中一个；
- 任一计数器达到上限后必须进入 `abstain`、`human_interrupt` 或 `fail`；
- 超时、取消和模型/索引不兼容必须具有显式终止边。

第二轮本地检索只能执行预先定义的纠正动作，例如扩大候选窗口、修正 metadata filter、执行 Planner 子查询或使用一次允许的 Query Rewrite。它不能切换到 Fast A Baseline，也不能继续生成新的 Rewrite 循环。

## 题型路由契约

`question_router` 至少支持以下稳定枚举：

- `simple_fact`：简单事实、定义、单点查询；
- `ambiguous_context`：存在可由上下文补全或必须询问的歧义；
- `comparison`：多个对象或条件对比；
- `causal`：原因、影响或机制问题；
- `multi_hop`：需要跨来源或多个子问题；
- `current`：含“当前、最新、今天、现行”等时效需求；
- `structured_numeric`：应由 SQL/Data API 返回的结构化字段或统计值；
- `out_of_scope`：当前知识源和允许工具无法支持。

题型可以影响路径，但不能绕过 Evidence Grade、Conflict Resolution、Grounding 和 Citation。路由输出只包含结构化标签与简短 `reason_code`，不保存 Chain of Thought。

## Checkpoint、Run 隔离与幂等

- LangGraph Checkpointer 固定使用 PostgreSQL 持久化；开发和生产都不得以进程内 MemorySaver 作为正式实现；
- `run_id` 用作 LangGraph `thread_id`，保证每次提问和 Human Interrupt 拥有独立执行状态；
- `conversation_id` 只作为业务会话标识保存在 State，用于 `load_context`、消息归属和同会话调度；
- Human Interrupt 恢复必须使用原 `run_id/thread_id`，不能创建新 thread；
- `checkpoint_ns` 或等价命名空间记录 `graph_version`，Graph State Schema 不兼容时不得静默恢复旧 Checkpoint；
- Checkpoint 负责 Graph 恢复，SQL 消息和 ChatRun 表负责业务真值，二者不能互相替代；
- Tool 调用、Web Search、消息写入、运行持久化和记忆写入都需要幂等键；
- 恢复时跳过已经成功且结果仍有效的副作用 Node；
- `waiting_user` 保存中断问题、允许回答格式和恢复目标；
- Checkpoint PostgreSQL Schema、迁移、连接池、保留和清理由 `backend/database/` 管理，Graph 只通过 Checkpointer 接口读写；
- Checkpointer 不可用或版本不兼容时必须明确失败，不得切换到内存状态后继续；
- Checkpoint 不对前端、日志或普通 API 暴露。

## `astream` 事件

Graph 可以产生内部事件，但必须经过 Event Adapter 才能成为 SSE：

- 可公开：`status`、`plan_summary`、`retrieval`、`tool_started`、`tool_finished`、`evidence`、`conflict`、`interrupt`、最终 `token`、`citation`、`completed`、`error`；
- 不公开：完整 State、Prompt、Checkpoint、隐藏推理、内部打分细节、未脱敏参数和 `answer_candidate`；
- `astream` 是执行与答案流式传输机制，不是 FLARE 式边生成边检索；
- 在 Grounding 结束前可以持续发送状态事件，但不能发送候选答案 Token；`emit_answer` 发送的是已冻结文本，因此不是模型原生生成时 Token 流。

## 非目标与未来扩展

当前 Graph 不实现 FLARE、Self-RAG、RAPTOR、GraphRAG、多 Agent 或多轮反思。未来新增层级检索时，应通过标准 Retriever 接口返回相同 Evidence Schema，不改变题型路由、证据检查、引用和终止条件，也不得丢失摘要节点到原始 Chunk 的可追溯关系。

## 最小验收场景

未来实现至少覆盖：

1. 清晰简单问题只走一次本地检索；
2. 可补全歧义只改写一次再检索；
3. 关键歧义进入 Human Interrupt 并可从 Checkpoint 恢复；
4. 对比/多跳问题最多产生三个子问题；
5. 本地证据不足时最多进行第二轮本地检索；
6. 最新信息在满足策略时最多执行一次 Web Search；
7. 结构化数值问题只调用白名单 Tool；
8. 冲突资料被并列说明或进入 Human Interrupt；
9. 最终候选中的无证据关键 Claim 导致拒答或中断，不进入对外答案；
10. 循环上限、超时、取消、Checkpoint 恢复和幂等副作用均可验证；
11. Thinking 全路径不会加载 Fast A Baseline；
12. 同一 Conversation 的下一次 Run 使用全新计数器、Plan、Evidence 和 Candidate；
13. SSE 中不存在隐藏推理、候选答案、Prompt、完整 State 或密钥。
