# CCAI Chatbot 实施与验收清单

本文档是后续编码 Agent 的执行说明和项目验收清单。目标是在当前仓库中逐步完成一个模块化单体 Chatbot：

- 后端：FastAPI、SQL、Redis、Celery；
- RAG：LangChain、LangGraph；
- 模型：PyTorch 训练后的 Chunker、Embedding 和 Cross-Encoder Reranker；
- 在线模式：Fast 与 Thinking；
- Fast：允许 Baseline/Trained A/B Test；
- Thinking：只使用训练版模型和索引；
- 交互：SSE + LangGraph `astream` 实时答案与执行进度；
- 能力：Checkpoint、记忆、引用、Web Search、Human-in-the-loop、外部工具接口；
- 评估：速度、召回、排序、答案质量、忠实性、引用和任务成功率。

本项目不采用微服务，先完成可维护、可测试的模块化单体。

---

## 1. Agent 工作规则

### 1.1 每次开始前

- [ ] 阅读本文件以及目标目录内所有 `readme.md`。
- [ ] 运行 `git status --short`，识别并保留用户已有修改。
- [ ] 明确本次只实现哪一个或哪几个相邻复选项，不做无关重构。
- [ ] 找出该功能涉及的 API、Service、Repository、RAG、数据集和测试路径。
- [ ] 先写出本次验收方法，再开始修改代码。

### 1.2 每完成一个小部分

- [ ] 运行最窄的相关测试或可执行验证。
- [ ] 确认错误路径、超时、空输入和资源释放行为。
- [ ] 更新对应目录 README，记录职责、输入输出和运行方式。
- [ ] 只有代码、测试、文档和验证全部通过后，才把对应任务从 `- [ ]` 改为 `- [x]`。
- [ ] 在该任务后追加一行完成记录：日期、修改文件、验证命令、验证结果和生成制品路径。
- [ ] 不得为了让清单看起来完成而提前勾选，不得删除失败记录。

完成记录格式：

```text
完成记录：YYYY-MM-DD HH:MM TZ；文件：...；验证：...；结果：PASS/FAIL；制品：...
```

### 1.3 实施边界

- [ ] 不引入微服务、消息总线或分布式部署，除非用户以后明确要求。
- [ ] 当前不实现鉴权；接口边界要保留以后增加鉴权的空间。
- [ ] 不实现多 Agent、Deep Research、多轮反思、无限 Query Rewrite 或无限工具探索。
- [ ] Query Rewrite 最多一次，本地检索最多两轮，Web Search 最多一次，外部工具调用设置明确上限。
- [ ] Bad Case 只记录、分类和形成回归样本，不进行无界自动探索。
- [ ] API Key、数据库密码和模型服务密钥只能从环境或密钥系统读取，不得提交到 Git。
- [ ] 当前训练层为 interface-only：未经用户再次明确授权，不创建训练函数、Dataset Builder、train/dev/test 数据、Checkpoint 或训练运行。
- [ ] 未来 Embedding/Cross-Encoder 固定基于官方 `huggingface/sentence-transformers` GitHub 源码，Chunker 固定基于 `mirth/chonky` ModernBERT 路线。
- [ ] 当前入口限流、安全 Header 和全局模型/SSE 容量控制为 interface-only；但同一 Conversation 单活动 ChatRun、会话 Dispatcher、短协调锁和乐观锁属于必须实现的正确性能力。

---

## 2. 不能违反的产品规则

### 2.0 生成式 LLM

- [ ] 线上生成式 LLM 固定使用 DeepSeek API，具体模型 ID 由配置明确登记。
- [ ] DeepSeek API 用于答案生成、Query Rewrite、路由、Planner、Evidence Grade、Grounding、摘要和记忆抽取等 LLM 能力。
- [ ] DeepSeek 不替代 OpenAI Embedding、训练 Embedding、BM25、Cross-Encoder 或本地 PyTorch 模型。
- [ ] Fast A、Fast B 的 A/B Test 必须使用相同 DeepSeek 模型 ID、Prompt 和生成参数。
- [ ] DeepSeek Provider 不可用时明确失败，不得静默切换到其他生成模型。
- [ ] DeepSeek 的隐藏推理或 reasoning content 不得写入 SSE、日志、Graph State 或 Checkpoint。

### 2.1 Fast 模式

- [ ] Fast A 使用 Semantic Chunking + 对应 BM25 + OpenAI Embedding KNN + RRF + Baseline Cross-Encoder。
- [ ] Fast B 使用训练 Chunking + 对应 BM25 + 训练 Embedding KNN + RRF + 训练 Cross-Encoder。
- [ ] 只有 Fast 模式参加 A/B Test。
- [ ] A、B 使用相同冻结测试集、生成模型、Prompt、最终上下文容量、硬件条件和指标。
- [ ] A、B 分别建立索引，不能跨向量空间复用索引。
- [ ] A/B 报告只说明完整系统差异，不声称某个单组件造成提升。

### 2.2 Thinking 模式

- [ ] Thinking 只加载登记过的训练版模型和索引。
- [ ] Thinking 不参加 A/B Test，也不生成 A/B 分组。
- [ ] Thinking 不得调用 Semantic Chunking + OpenAI Embedding 的 Baseline。
- [ ] 训练模型或索引不可用时明确失败或进入可恢复流程，不得静默回退到通用 RAG。
- [ ] Thinking 保留 Planner、Orchestrator、Checkpoint、Web Search、长期记忆、Human-in-the-loop 和外部工具接口。
- [ ] 没有 Active trained Snapshot 时 `THINKING_ENABLED=false`，不得把 Test Double 或 Fast A 作为生产回退。

### 2.3 实时输出与 Chain of Thought

- [ ] 使用 `astream`/`astream_events` 实现实时输出，但不得向用户暴露模型私有 Chain of Thought。
- [ ] 前端可以看到答案 Token、节点状态、简短计划摘要、检索数量、工具开始/结束、引用和中断状态。
- [ ] 前端不得看到隐藏推理、原始 Prompt、完整 Graph State、Checkpoint 内容、调试事件、密钥或未脱敏工具参数。
- [ ] 产品文案使用“实时执行轨迹”或“执行进度”，不声称展示模型内部思维链。

### 2.4 同会话并发控制与延期能力

当前必须实现：

- 所有持久化记录使用数据库生成的 UTC `created_at`；
- 所有可变记录增加数据库生成的 `updated_at` 和整数 `version`；
- 更新使用 `id + expected_version` compare-and-swap，成功时原子 `version + 1`；
- Message 使用唯一 `client_message_id`，并使用 `conversation_id + sequence_number` 保证会话顺序；
- 同一用户的不同 Conversation 可以并行；同一 Conversation 的 Fast/Thinking ChatRun 严格串行；
- 同一 Conversation 可以有多个 Pending Run，但同时最多一个 `running` 或 `waiting_user` Run；
- `waiting_user` 继续占用会话执行槽，恢复或取消后才能调度下一条；
- SQL 保存队列和活动 Run 的最终状态；Redis 短锁只覆盖 Dispatcher 的短临界区；
- 当前 Run 完成、失败或取消后，按 `sequence_number` 调度下一条 Pending Run；
- 乐观锁、唯一约束、消息幂等和并发请求必须具有测试。

当前继续延期：入口限流、CSP/HSTS 等安全 Header、全局请求/SSE/模型 Semaphore 和跨会话容量控制。后续 Agent 不得把这些延期能力描述为已启用。

### 2.5 Retrieval 与 Rerank 强制架构

- [ ] Sparse Retrieval 固定使用版本锁定的 `bm25s` 实现真正 BM25；PostgreSQL `ts_rank/ts_rank_cd` 不得被标记为 BM25。
- [ ] Dense Retrieval 固定使用 PostgreSQL + pgvector；离线正确性基准使用 Exact KNN，在线生产使用 HNSW ANN。
- [ ] Query 同时执行 BM25 Top-K 与 Dense KNN Top-K，再使用 RRF 合并排名；禁止直接相加 BM25 Score 和向量相似度。
- [ ] RRF 后统一去重、限制每个来源的 Chunk 数，再交给 Cross-Encoder Reranker。
- [ ] Hybrid Result 使用 `retrieval_hits[]` 分别保存 BM25/Dense 的 raw score 和 rank，不使用单个字段覆盖双路命中。
- [ ] 去重和相邻 Chunk 合并保留 leaf_chunk_ids、source_spans、document_version、offset 和 content_hash，Citation 必须可回到原文。
- [ ] Fast A 使用 Semantic Chunk、OpenAI Embedding Dense Index、对应 BM25 Index 和 Baseline Cross-Encoder。
- [ ] Fast B 使用训练 Chunk、训练 Embedding Dense Index、对应 BM25 Index 和登记的训练 Cross-Encoder。
- [ ] Thinking 复用训练版 Chunk、BM25/Dense Index 和训练 Cross-Encoder；不得使用 Fast A 的 Chunk、向量索引或 Baseline Reranker。
- [ ] BM25 是基于训练 Chunk 构建的确定性词法算法，不属于“通用 RAG 模型回退”，因此 Thinking 可以使用训练版 Snapshot 内的 BM25。
- [ ] 每个 Retrieval Snapshot 不可变，并记录 Corpus、Chunker、Embedding、BM25 Analyzer、Reranker、参数和文件哈希。
- [ ] BM25 与 Dense 的索引构建、验证和发布由 Celery 完成；在线请求只读取 ACTIVE Snapshot，离线验收才可以显式读取 READY Snapshot。
- [ ] Snapshot 按 `environment + snapshot_family(fast_a|trained)` 隔离，数据库保证每个家族最多一个 Active。
- [ ] Snapshot 激活使用短 Redis 协调锁和单个 SQL 事务完成 old active→retired、ready→active；失败时旧 Active 保持不变。
- [ ] 每个 ChatRun 启动时固定并持久化 snapshot_id，运行途中不得切换 BM25、Dense、Chunk 映射或 Reranker 版本。

---

## 3. 参考资料

实现可以参考以下教程的结构和概念，但必须重构为本项目模块，不能直接把 Notebook 当生产代码：

- [尚硅谷 LangGraph 课件](https://github.com/xbsheng/atguigu-note/tree/main/langgraph/%E8%AF%BE%E4%BB%B6)：重点参考持久化、Checkpoint、流式执行、Human-in-the-loop、子图、Routing 和 Orchestrator-worker。
- [LangChain OpenTutorial 固定版本](https://github.com/LangChain-OpenTutorial/LangChain-OpenTutorial/tree/151d715598bc2a5d3d751a6262a59878f54d2368)：重点参考 Naive RAG、Agentic RAG、Adaptive RAG、Plan-and-Execute、Query Rewrite、文档评分和 Streaming。
- [Agentic RAG 示例](https://github.com/LangChain-OpenTutorial/LangChain-OpenTutorial/blob/151d715598bc2a5d3d751a6262a59878f54d2368/17-LangGraph/02-Structures/06-LangGraph-Agentic-RAG.ipynb)：参考 retrieve、grade、rewrite、generate 的节点边界；必须增加循环上限。
- [Adaptive RAG 示例](https://github.com/LangChain-OpenTutorial/LangChain-OpenTutorial/blob/151d715598bc2a5d3d751a6262a59878f54d2368/17-LangGraph/02-Structures/07-LangGraph-Adaptive-Rag.ipynb)：参考本地知识库与 Web Search 的条件路由。
- [Streaming 示例](https://github.com/LangChain-OpenTutorial/LangChain-OpenTutorial/blob/151d715598bc2a5d3d751a6262a59878f54d2368/17-LangGraph/01-Core-Features/15-LangGraph-Streaming-steps.ipynb)：参考节点更新和消息流；对外事件必须经过过滤。
- [Plan-and-Execute 示例](https://github.com/LangChain-OpenTutorial/LangChain-OpenTutorial/blob/151d715598bc2a5d3d751a6262a59878f54d2368/17-LangGraph/03-Use-Cases/05-LangGraph-Plan-and-Execute.ipynb)：参考 Planner 与执行器分离。

教程只能作为参考。最终实现必须具有项目自己的 Schema、错误处理、测试、配置、超时、重试、持久化和停止条件。

---

## 4. 目标目录边界

- [x] Backend 已按 `fastapi / database / task / service` 完成二级分类。
  - 完成记录：2026-09-26 HKT；文件：`backend/` 目录与 README；验证：目录检查、旧路径检查、`git diff --check`；结果：PASS；制品：无。
- [x] RAG 已按 `online / pipeline / orchestration / knowledge / integration` 完成二级分类。
  - 完成记录：2026-09-26 HKT；文件：`rag/` 目录与 README；验证：目录检查、旧路径检查、`git diff --check`；结果：PASS；制品：无。
- [x] Fast A/B Test 文档已建立，并明确 Thinking 不参加 A/B。
  - 完成记录：2026-09-26 HKT；文件：`test/test-rag/`；验证：README 路径与规则检查、`git diff --check`；结果：PASS；制品：无。
- [x] Backend 空 README 已补齐职责、内容和分层边界。
  - 完成记录：2026-09-28 10:43:53 HKT；文件：`backend/config/readme.md`、`backend/fastapi/handler/readme.md`、`backend/fastapi/middleware/readme.md`、`backend/fastapi/schema/readme.md`、`backend/util/readme.md`；验证：非空断言、边界关键词检查、`git diff --check -- backend`；结果：PASS；制品：无。
- [x] Train 上游来源、预留接口和当前禁止实现边界已建立。
  - 完成记录：2026-09-28 10:47:54 HKT；文件：`train/agent.md`、`agent.md`、`rag/integration/model_runtime/readme.md`、`test/test-rag/shared-test-set/README.md`；内容：Embedding/Cross-Encoder 固定参考官方 Sentence Transformers GitHub，Chunker 固定参考 Chonky ModernBERT，当前只保留 Job/Artifact/Dataset/Runtime 契约；验证：Train 非 Markdown 文件检查、Dataset/Test 数据文件检查、上游 URL/Commit/禁用边界搜索、`git diff --check`；结果：PASS；制品：无。
- [x] Thinking 的有界纠错、题型路由和 LangGraph 条件边文档已建立；这只表示架构说明完成，不表示代码已实现。
  - 完成记录：2026-09-28 10:56:28 HKT；文件：`rag/online/thinking/readme.md`、`rag/orchestration/graph/readme.md`、`rag/orchestration/schema/readme.md`、`rag/orchestration/prompt/readme.md`、`agent.md`；内容：增加 Input Guard、Question Router、受限 Corrective Retrieval、Conflict Resolution、一次 Answer Grounding、Human Interrupt、Checkpoint/幂等、SSE 安全边界和明确非目标；验证：关键契约搜索、未提前勾选实现项检查、尾随空白检查、`git diff --check`；结果：PASS；制品：无。
- [x] 限流、安全 Header 和后端并发控制已移出当前实现范围，只保留未来接口契约。
  - 完成记录：2026-09-28 15:19:54 HKT；文件：`agent.md`、`backend/config/readme.md`、`backend/fastapi/middleware/readme.md`、`backend/fastapi/schema/readme.md`、`backend/database/readme.md`、`backend/database/model/readme.md`、`backend/database/repository/readme.md`、`backend/database/redis/readme.md`、`backend/service/readme.md`、`rag/knowledge/memory/readme.md`、`rag/pipeline/retrieval/readme.md`、`test/test-rag/metrics/README.md`；内容：统一标记为 DEFERRED/interface-only，禁止当前实现 Redis 限流、安全 Header 注入、运行时 Semaphore、会话 Dispatcher、分布式锁和乐观锁执行；保留配置命名空间、接口名、字段、状态和错误类型；验证：延期边界关键词检查、强制实现项复核、尾随空白检查、`git diff --check`；结果：PASS；制品：无。
- [x] 同一 Conversation 单活动 ChatRun 的并发控制架构已重新启用，并完成其余架构一致性修订；这只表示文档计划完成，不表示代码已实现。
  - 完成记录：2026-09-28 16:04:48 HKT；文件：`README.md`、`agent.md`、`backend/config/readme.md`、`backend/fastapi/middleware/readme.md`、`backend/fastapi/schema/readme.md`、`backend/database/readme.md`、`backend/database/model/readme.md`、`backend/database/repository/readme.md`、`backend/database/redis/readme.md`、`backend/service/readme.md`、`backend/task/celery/readme.md`、`dataset/sql/readme.md`、`frontend.md`、`frontend/readme.md`、`rag/integration/readme.md`、`rag/integration/model_runtime/readme.md`、`rag/integration/tool/readme.md`、`rag/knowledge/citation/readme.md`、`rag/knowledge/data_source/readme.md`、`rag/knowledge/evidence/readme.md`、`rag/knowledge/memory/readme.md`、`rag/online/readme.md`、`rag/online/fast/readme.md`、`rag/online/thinking/readme.md`、`rag/orchestration/graph/readme.md`、`rag/orchestration/schema/readme.md`、`rag/pipeline/chunking/readme.md`、`rag/pipeline/embedding/readme.md`、`rag/pipeline/retrieval/readme.md`、`test/readme.md`、`test/test-rag/README.md`、`test/test-rag/metrics/README.md`、`test/test-rag/reports/README.md`；内容：以 SQL 状态和数据库唯一约束保证每个 Conversation 最多一个 `running/waiting_user` Run，Redis 只做短协调锁；补齐消息幂等与乐观锁、Checkpoint 按 Run 隔离、先 Grounding 后答案流、Snapshot 原子发布与固定、Evidence/Citation Provenance、Tool/DataSource 边界、训练制品启用门槛、仅离线 Fast A/B 以及 Chunk/Embedding/SQL Dataset/Frontend/Test 文档；本决策覆盖上一条记录中“后端并发控制延期”的部分，入口限流、安全 Header 和全局容量 Semaphore 仍延期；验证：架构关键词与旧术语搜索、非空文档检查、尾随空白检查、`git diff --check`；结果：PASS；制品：无。
- [x] LangGraph Checkpoint 后端已固定为 PostgreSQL；这只表示架构决策完成，不表示 Checkpointer 代码或表已创建。
  - 完成记录：2026-09-28 16:10:01 HKT；文件：`README.md`、`agent.md`、`backend/config/readme.md`、`backend/database/readme.md`、`backend/database/model/readme.md`、`rag/orchestration/graph/readme.md`；内容：取消开发阶段正式使用内存 Checkpointer，规定 PostgreSQL 专用 Schema、受控连接池、显式迁移、`thread_id=run_id`、版本命名空间、事务边界、敏感/大对象限制、保留清理、不可用时明确失败和隔离 PostgreSQL 集成测试；验证：Checkpoint 旧表述搜索、关键契约搜索、尾随空白检查、`git diff --check`；结果：PASS；制品：无。

后续代码必须遵守：

```text
backend/
├── fastapi/       HTTP、SSE、Pydantic、Depends、异常与中间件
├── database/      ORM、Repository、Redis、Session、事务与迁移
├── task/          Celery 异步任务
├── service/       应用用例和跨模块编排
├── config/        配置
└── util/          无业务状态工具

rag/
├── online/        Fast 与 Thinking 入口
├── pipeline/      Chunking、Embedding、Retrieval、Rerank、Generation
├── orchestration/ Chain、Graph、Prompt、内部 Schema
├── knowledge/     Data Source、Evidence、Citation、Memory
└── integration/   Model Runtime、Tool、MCP/外部接口
```

---

## 5. 阶段 0：仓库基线与开发规范

- [ ] 确认 Python 版本和虚拟环境策略，并在 README 中固定。
- [ ] 整理依赖清单，区分运行、训练和开发测试依赖。
- [ ] 建立配置加载方式，覆盖本地、测试和生产环境。
- [ ] 提供 `.env.example`，只包含变量名和说明，不包含真实密钥。
- [ ] 建立日志格式，至少包含 request_id、conversation_id、run_id、mode 和耗时。
- [ ] 建立格式检查、静态检查和测试命令。
- [ ] 为主要包补充最小导入测试，确保目录结构可被 Python 正确加载。
- [ ] 更新根 README，给出项目结构、安装、启动和测试入口。

验收：全新环境可以安装依赖；配置缺失时给出明确错误；测试命令能够执行。

---

## 6. 阶段 1：SQL、ORM 与 Repository

- [ ] 在 `backend/database/` 建立 SQL Engine、Session Factory、连接池和请求级事务。
- [ ] 建立数据库健康检查和测试数据库隔离策略。
- [ ] 配置迁移工具，禁止依赖应用启动时删除重建表。
- [ ] 建立 ORM Base 和 ID 约定；所有持久化记录统一使用数据库 UTC `created_at`。
- [ ] 为所有可变记录统一增加 `updated_at`、从 1 开始的整数 `version` 和 `expected_version` 更新参数。
- [ ] Repository 更新统一执行 `WHERE id = ? AND version = expected_version`，成功后原子 `version + 1`。
- [ ] 定义乐观锁冲突异常、有限重读重试和失败返回，不使用 `updated_at` 代替版本号。
- [ ] 建立 Conversation 表。
- [ ] Conversation 保存当前滚动摘要指针，并通过 `version` 防止并发覆盖。
- [ ] 建立 Message 表；消息只追加，使用唯一 `client_message_id` 和 `conversation_id + sequence_number`。
- [ ] 建立 ChatRun 表，记录 Fast/Thinking、状态、队列顺序、延迟和错误。
- [ ] 用数据库部分唯一索引或等价约束保证同一 Conversation 同时最多一个 `running` 或 `waiting_user` ChatRun。
- [ ] 建立 Document、DocumentChunk 和 IndexSnapshot 表。
- [ ] 建立 Task 表，对应 Celery 任务生命周期。
- [ ] 建立 RollingSummary 表；新摘要追加新版本，保存覆盖消息范围和 `previous_summary_id`。
- [ ] 建立 LongTermMemory 与 MemoryRevision；逻辑事实 Upsert，物理修改追加 Revision。
- [ ] 建立 ModelVersion/ModelRegistry 表或等价持久化清单。
- [ ] 建立 Experiment、EvaluationRun、EvaluationResult 表。
- [ ] 为每个聚合建立 Repository 接口和实现。
- [ ] 对 Repository 的事务、分页、空结果、唯一冲突、乐观锁冲突和回滚进行测试。
- [ ] 测试两个事务同时更新 Conversation、ChatRun、Task 和当前摘要指针时不会静默覆盖。
- [ ] 如使用 pgvector，验证扩展、维度、距离函数和索引创建。

验收：迁移可从空数据库执行；Repository 集成测试通过；测试不会连接或清空生产数据库。

---

## 7. 阶段 2：Redis 与 Celery 基础设施

- [ ] 建立 Redis 连接、健康检查、超时和关闭流程。
- [ ] 统一 Redis Key 命名、TTL 和序列化规范。
- [ ] Redis 当前只保存缓存、A/B 分组、短期状态和 Celery Broker 数据。
- [ ] 为未来 `RateLimitStore` 和全局 `ConcurrencyLease` 保留接口，默认不启用。
- [ ] 建立 Redis 分布式短锁封装：唯一 owner token、TTL、原子比较后释放和有限等待。
- [ ] Redis 短锁用于 Conversation Dispatcher、Snapshot 构建/激活、摘要调度、记忆整理和评估去重。
- [ ] 不使用覆盖整个 LLM、SSE 或 LangGraph 执行周期的 Redis 长锁。
- [ ] Redis 锁不是唯一一致性保障；关键写入同时使用 SQL 事务、唯一约束、状态条件或乐观锁。
- [ ] 建立 Celery App、队列、任务路由和 Worker 启动说明。
- [ ] 建立任务幂等键、超时、重试和失败状态写回。
- [ ] 建立文档处理任务骨架。
- [ ] 建立索引构建任务骨架。
- [ ] 建立 Fast A/B 离线评估任务骨架。
- [ ] 验证 Worker 不可用、任务超时和重复提交行为。

验收：测试任务可以提交、执行、查询状态和失败重试；实时聊天不依赖 Celery 才能返回。

---

## 8. 阶段 3：FastAPI 外壳

- [ ] 建立 FastAPI Application Factory 和生命周期管理。
- [ ] 将数据库、Redis、RAG Runtime 的启动与关闭接入生命周期。
- [ ] 建立请求 ID 和结构化日志 Middleware。
- [ ] 为 `RateLimiter`、`SecurityHeaderPolicy` 和全局 `RuntimeCapacityController` 保留 Middleware/Dependency 插入位置，但不实现运行逻辑。
- [ ] 当前不得添加 Redis 限流算法、CSP/HSTS 等安全 Header 中间件或全局请求/SSE/模型 Semaphore；同会话并发控制由 Service/SQL 实现。
- [ ] 建立统一错误响应 Schema 和异常处理器。
- [ ] 建立健康检查、就绪检查和版本接口。
- [ ] 建立 Conversation Router。
- [ ] 建立 Message/History Router。
- [ ] 建立 Document/Task Router。
- [ ] 建立 Chat Router，但先使用可替换的 Stub Service。
- [ ] 建立 SSE Chat Router 的连接、心跳、取消和断线清理骨架。
- [ ] 确保 Handler 只处理 HTTP/SSE，不写 SQL、不拼 Prompt、不实现 RAG。
- [ ] 为每个接口编写成功、校验失败、资源不存在和内部错误测试。

验收：API 文档可访问；Handler 测试通过；断开 SSE 后资源被释放。

---

## 9. 阶段 4：数据集、切分与索引数据契约

当前状态：仅保留下列未来验收项，不创建任何 train/dev/test 文件。只有用户明确授权数据构建后才执行。

- [ ] 定义原始文档、规范化文档、Chunk、Query、Qrels 和参考答案 Schema。
- [ ] 保留 source_id、URL、许可、版本、抽取质量和审核状态。
- [ ] 按来源文档或明确语义组隔离 train/dev/test。
- [ ] 检查 source overlap、exact-text overlap、normalized-text overlap 和 query overlap。
- [ ] 使用实际 Tokenizer 检查长度，不使用空格词数替代。
- [ ] 自动章节、布局、语义或边界标签一律标记为弱监督。
- [ ] 未经独立人工审核的测试集不得称作 Golden Test。
- [ ] 冻结测试集版本、文件哈希、样本数和 Qrels 数量。
- [ ] 建立文档导入、规范化和失败清单。
- [ ] 建立索引 Manifest，记录 Corpus、Chunker、Embedding、向量维度、距离函数、BM25 参数、Analyzer、RRF 和 Reranker。
- [ ] 建立稳定 `doc_index → chunk_id → document_id/source_id` 映射，重建索引后不得错配正文或来源。
- [ ] 定义 BM25 Analyzer 契约：Unicode 规范化、语言识别、中英文分词、数字/年份保留、停用词和领域词典版本。
- [ ] 索引与查询必须使用相同 Analyzer 版本；版本不匹配时拒绝加载。
- [ ] 定义不可变 IndexSnapshot 状态：building、validating、ready、active、retired、failed。
- [ ] 定义 Snapshot 家族、原子激活、回滚和“每个 Run 固定 snapshot_id”契约。

验收：审计脚本能够阻止切分泄漏、缺失来源、无效引用和超长输入。

---

## 10. 阶段 5：通用 Baseline Fast RAG

- [ ] 在 `rag/pipeline/chunking/` 封装 Semantic Chunking。
- [ ] Chunk 输出保留 document_version、稳定 offset、content_hash、Tokenizer token_count 和确定性 chunk_id。
- [ ] 记录语义断点算法、阈值、Tokenizer、最小/最大长度和后备切分。
- [ ] 在 `rag/pipeline/embedding/` 封装 OpenAI Embedding。
- [ ] Embedding 契约固定 input_type、模型/前缀版本、维度、归一化、最大长度、距离函数和截断信息。
- [ ] 记录模型 ID、维度、归一化、批量、超时、重试和成本。
- [ ] 建立文档 Embedding 缓存，缓存键包含文本哈希和模型版本。
- [ ] 在 `rag/pipeline/retrieval/` 建立 Sparse、Dense 和 Hybrid Retriever 统一接口。
- [ ] 使用固定版本 `bm25s` 构建 Fast A BM25 Index；保存索引、Chunk 映射和 Manifest。
- [ ] 使用 OpenAI Embedding + pgvector 构建 Fast A Dense Index。
- [ ] 离线评估实现 Exact KNN；在线实现 HNSW，并记录距离函数、M、ef_construction 和 ef_search。
- [ ] BM25 与 Dense 并发召回，各自返回独立 rank、score、retrieval_source 和 snapshot_id。
- [ ] 使用 RRF 融合两路排名，不对原始分数直接加权求和。
- [ ] RRF 后执行 Chunk 去重、重叠文本去重、相邻 Chunk 合并和每来源数量上限。
- [ ] 建立 metadata filter、top_k、过滤后补足和空结果处理；硬过滤不能只靠召回后的偶然筛选。
- [ ] 在 `rag/pipeline/rerank/` 建立 Cross-Encoder Rerank 接口。
- [ ] Fast A 使用登记的 Baseline Cross-Encoder，固定候选窗口和最终 Top-N Evidence。
- [ ] 记录 Rerank 前后 rank、score、模型版本和延迟。
- [ ] 在 `rag/pipeline/generation/` 建立基于证据的答案生成。
- [ ] 通过统一 Provider 契约调用配置中登记的 DeepSeek API，不在 Pipeline、Graph 或 Prompt 中直接创建客户端。
- [ ] DeepSeek Provider 支持超时、有限重试、取消、流式与非流式调用、结构化输出校验、用量和延迟记录。
- [ ] 建立证据不足时的拒答，不允许只靠模型常识补写。
- [ ] 建立 Citation 输出。
- [ ] 组装 Fast A Pipeline。
- [ ] 编写固定语料的检索和端到端测试。
- [ ] 测量冷/热缓存下 Embedding、Retrieval、Rerank、TTFT 和总延迟。

验收：Fast A 能从冻结语料回答、拒答和返回引用；每阶段延迟可观测。

---

## 11. 阶段 6：PyTorch 训练数据与训练流程

当前状态：**DEFERRED**。只允许维护 `train/agent.md` 中的接口契约；未经用户再次明确授权，下面所有训练、数据和测试项都不得实施。

固定上游来源：

```text
Embedding / Cross-Encoder:
https://github.com/huggingface/sentence-transformers.git

Chunking:
https://github.com/mirth/chonky.git
ModernBERT Base 为默认候选，Large 为独立可选版本
```

### 11.1 Chunker

- [ ] 用户已明确解除 Chunker 训练实现限制。
- [ ] Chonky GitHub Commit、ModernBERT Model ID、License 和语言覆盖已登记。
- [ ] 定义 Chunking/边界模型训练输入输出。
- [ ] 构建 source-disjoint train/dev/test。
- [ ] 弱标签与人工标签分开保存。
- [ ] 禁止把直接决定标签的元数据传入模型。
- [ ] 使用实际 Tokenizer 验证训练序列长度。
- [ ] 实现训练、验证、Checkpoint 和 Resume。
- [ ] 每次训练写入新输出目录，不覆盖旧模型。
- [ ] 记录 seed、数据版本、基础模型、阈值和超参数。
- [ ] 使用二元 Precision、Recall、F1、TP/FP/FN/TN 和 Average Precision 评估边界。
- [ ] 保存训练报告、最佳 Checkpoint 和失败样本。

### 11.2 Embedding

- [ ] 用户已明确解除 Embedding 训练实现限制。
- [ ] Sentence Transformers GitHub Commit、Base Model、License 和训练接口版本已登记。
- [ ] 定义 Query、Positive、Negative 训练格式。
- [ ] 构建显式负例和难负例，并排除可能的假负例。
- [ ] 防止同一 Positive 的不同 Query 在同批中成为假负例。
- [ ] 记录训练数据来源和弱监督状态。
- [ ] 实现训练、验证、Checkpoint 和 Resume。
- [ ] 每次训练写入新输出目录。
- [ ] 在独立 dev 集报告 Recall@K、MRR@K 和 nDCG@K。
- [ ] 不使用冻结 test 调参或选择阈值。
- [ ] 保存模型、Tokenizer、配置和训练报告。

### 11.3 Reranker

- [ ] 用户已明确解除 Reranker 训练实现限制。
- [ ] Sentence Transformers CrossEncoder GitHub Commit、Base Model 和标签定义已登记。
- [ ] 在 Chunker/Embedding 基线稳定后训练领域 Cross-Encoder Reranker。
- [ ] 定义 Pairwise 或 Pointwise 数据和标签含义。
- [ ] 记录候选生成方式，防止训练/测试泄漏。
- [ ] 报告 Rerank 前后 Recall、MRR、nDCG 和延迟变化。
- [ ] 保存训练 Reranker 的模型、Tokenizer、最大长度、阈值、数据版本和训练报告。
- [ ] Fast B 与 Thinking 正式启用前，训练 Reranker 必须进入 Model Registry；不可用时明确失败，不静默回退 Baseline Reranker。

验收：训练可复现；模型和报告可追溯到数据版本；测试数据从未参与调参。

---

## 12. 阶段 7：Model Registry 与 Model Runtime

- [ ] 定义 Model Registry 条目：model_id、类型、版本、Checkpoint、Tokenizer、数据版本和状态。
- [ ] 定义 candidate、staging、production、retired 状态。
- [ ] 在 `rag/integration/model_runtime/` 建立 Chunker 加载与运行接口。
- [ ] 建立 Embedding 加载与运行接口。
- [ ] 建立训练 Cross-Encoder Reranker 的加载、批处理、评分和关闭接口。
- [ ] 统一 `load / warmup / run / health / close` 生命周期。
- [ ] 支持 CPU、MPS/CUDA、精度、批处理和最大长度配置。
- [ ] 检查 Checkpoint 与 Tokenizer/维度不兼容。
- [ ] 记录模型加载时间、推理延迟和峰值内存。
- [ ] 建立资源释放和 Model Runtime 基本并发安全测试；全局容量 Semaphore 当前延期。
- [ ] Thinking Runtime 禁止通用 Baseline 回退。
- [ ] 提供仅测试依赖注入可用的确定性 Runtime/Retriever Test Double，禁止进入生产 Registry 或 Snapshot。

验收：给定 Registry 版本可确定性加载模型；错误版本明确失败；进程关闭后资源释放。

---

## 13. 阶段 8：训练版 Fast RAG 与 A/B Test

- [ ] 使用训练 Chunker 构建 Fast B/Thinking 共用的 BM25 IndexSnapshot。
- [ ] 使用训练 Embedding + pgvector 构建 Fast B/Thinking Dense IndexSnapshot。
- [ ] 使用相同 Hybrid 流程：BM25 + Dense KNN → RRF → 去重/来源限制 → 训练 Cross-Encoder。
- [ ] 使用训练 Chunker、训练 Embedding、Hybrid Retrieval 和训练 Reranker 组装 Fast B。
- [ ] Fast B 使用独立 IndexSnapshot。
- [ ] 没有 Active trained Snapshot 时 `FAST_B_ENABLED=false`；只有 Fast A 可以在线运行。
- [ ] 第一阶段只实现 Offline Paired Benchmark，每个 Query 同时运行 A/B，不进行真实用户流量分组。
- [ ] Shadow 模式与 Live Traffic A/B 作为以后独立阶段；Live A/B 未经用户授权不得实现。
- [ ] 未来 Live A/B 才定义稳定 Hash Assignment、Redis 缓存、SQL 实验记录和退出条件。
- [ ] A/B 只发生在 Fast；当前离线成对评估不建立在线 Assignment，Thinking 永远不能读取未来的 A/B Assignment。
- [ ] 建立离线成对评估，所有 Query 同时运行 A 和 B。
- [ ] `DEFERRED`：以后经单独决策才建立 Shadow 模式，由 A 返回用户、B 后台记录且不影响实时响应。
- [ ] `DEFERRED`：Shadow 任务必须设置采样率、成本上限和隐私规则。
- [ ] 建立实验版本、测试集版本、索引版本和模型版本记录。
- [ ] 报告逐 Query 差异和总体指标，不做组件消融结论。

验收：Offline Paired Benchmark 对相同 Query 运行 A/B 并可完整复现；Thinking 永远不参与；不得把离线结果描述成线上用户 A/B。

---

## 14. 阶段 9：Thinking LangGraph 基础

- [ ] 定义 Thinking Graph State，不直接复用 API Schema。
- [ ] State 至少包含 conversation/run 标识、original/effective query、input assessment、question type、route、freshness、plan、evidence、conflict、grounding、citations、计数器、interrupt、answer candidate 和 final answer。
- [ ] 实现 load_context Node。
- [ ] 实现 input_guard Node，区分 clear、auto_rewrite、need_user、incorrect_premise 和 invalid。
- [ ] Input Guard 检查主体、实体、时间、地点、单位、关键歧义和错误前提，不覆盖原始用户问题。
- [ ] 模糊问题只有在当前会话可可靠补全时，才由 AI 生成明确检索 Query 后再 Embedding。
- [ ] Query Rewrite 最多一次；同时保存 original_query 与 effective_query，改写保留主体、时间、地点、单位和用户约束。
- [ ] 实现结构化 question_router Node，输出 question_type、route、needs_freshness 和不含私有思维链的 reason_code。
- [ ] 题型至少覆盖 simple_fact、ambiguous_context、comparison、causal、multi_hop、current、structured_numeric 和 out_of_scope。
- [ ] 简单事实直接进入本地检索；对比、因果和多跳问题进入 Planner；最新信息先查本地再按证据时效性决定 Web；结构化数值问题进入白名单 SQL/Data API Tool。
- [ ] 实现结构化 Planner，最多生成三个子问题。
- [ ] 实现单 Orchestrator，不拆成多 Agent。
- [ ] 实现训练版 local_retrieve Node。
- [ ] local_retrieve 只能调用训练版 BM25 + 训练 Embedding pgvector KNN/HNSW + RRF + 训练 Cross-Encoder Snapshot。
- [ ] 实现 evidence_grade Node，检查相关性、覆盖度、可信度、时效性和可回答性。
- [ ] Evidence Grade 只能路由到 sufficient、retry_local、need_web、need_user 或 insufficient。
- [ ] Web Search 结果和 Tool 结果必须转换成统一 Evidence，并再次经过 Evidence Grade，不能直接进入 Generation。
- [ ] 实现 conflict_resolution Node。
- [ ] 实现内部 answer_generate_candidate Node，生成完整候选文本但不得通过 SSE 发送。
- [ ] 实现 bind_citations Node，将候选文本 Claim 绑定到实际进入上下文的 Evidence。
- [ ] 实现一次性 final_grounding Node，检查将要返回的完整文本、Claim、Evidence、Citation、无支撑事实和冲突呈现。
- [ ] Grounding 通过后不得再次调用 LLM；Grounding 失败只能拒答或 Human Interrupt，不建立生成—反思—重写循环。
- [ ] 实现 emit_answer Node，只分块发送已经验证并冻结的文本，不生成或改写内容。
- [ ] 分离 persist_run 与 memory_candidate_write；拒答也保存运行记录，但不得自动成为长期记忆。
- [ ] 为所有条件边建立显式终止路径。
- [ ] 设置 recursion_limit、节点超时和业务计数器，不能只依赖 Graph recursion_limit。
- [ ] Query Rewrite 最多一次、本地检索最多两轮、Web Search 最多一次、Planner 最多三个子问题，Tool 调用使用显式上限。
- [ ] 上限耗尽后只能进入 abstain、human_interrupt 或 fail，不得静默回退 Fast A Baseline。
- [ ] `astream` 在 Grounding 前只公开安全执行轨迹；Grounding 后分块发送冻结答案，不得发送 Candidate、隐藏推理、Prompt、完整 State 或 Checkpoint。
- [ ] 当前不实现 FLARE 式边生成边检索、Self-RAG 反思 Token、RAPTOR、GraphRAG、多 Agent 或多轮反思。
- [ ] 为未来 hierarchical_retriever 保留标准 Retriever/Evidence 接口，但不得在未经独立评估时替换当前混合检索。
- [ ] 编写简单题、可改写歧义、必须询问歧义、错误前提、对比/多跳、最新信息、结构化 Tool、无结果、冲突、Grounding 失败、循环上限和 Checkpoint 恢复测试。

验收：所有 Graph 路径都能在明确上限内结束；路由结果可审计但不包含私有思维链；最终返回文本和 Citation 已被同一次 Grounding 检查且检查后未再生成；Thinking 全程只调用训练版模型与索引。

---

## 15. 阶段 10：Checkpoint 与恢复

- [ ] 开发、集成测试和生产统一接入 PostgreSQL Checkpointer；不得使用进程内 MemorySaver 作为正式实现。
- [ ] 使用独立 `langgraph_checkpoint` Schema；Checkpointer 内部表由锁定版本的 PostgreSQL Checkpointer 显式迁移管理，不建立重复业务 ORM。
- [ ] 配置独立连接池或池配额、连接获取超时、Statement Timeout 和健康检查，避免影响业务 SQL 连接。
- [ ] 使用 run_id 作为 LangGraph thread_id；conversation_id 只用于业务会话归属和加载上下文。
- [ ] Checkpoint 命名空间记录 graph_version/State Schema 版本，不兼容版本禁止静默恢复。
- [ ] 每个新 Run 初始化全新临时 State，不继承上一 Run 的计数器、Plan、Evidence、Conflict、Candidate 或 Tool Result。
- [ ] 明确业务消息表和 LangGraph Checkpoint 的边界。
- [ ] 测试节点失败后从 Checkpoint 恢复。
- [ ] 测试 Human Interrupt 使用原 run_id/thread_id 继续执行。
- [ ] 测试服务重启后的 Graph 恢复。
- [ ] 测试同一 Conversation 的下一次 Run 不读取上一 Run 的临时 Graph State。
- [ ] 建立 Checkpoint 保留、清理和隐私策略。
- [ ] 使用有界 Celery 任务分批清理已结束且超过保留期的 Checkpoint，不清理 waiting_user 或仍可恢复的 Run。
- [ ] Checkpointer 不可用时 Thinking 明确失败，不得静默切换 MemorySaver；纯单元测试 Test Double 不计入恢复验收。
- [ ] 在隔离 PostgreSQL 测试 Schema 验证服务重启、并发恢复、版本不兼容和清理行为。
- [ ] 不把完整 Checkpoint 暴露给前端或日志。

验收：中断与恢复不会重复写消息、重复调用工具或产生重复答案。

---

## 16. 阶段 11：`astream`、SSE 与实时执行轨迹

- [ ] 使用 LangGraph `astream` 或 `astream_events` 驱动 Thinking 流。
- [ ] Fast 模式也提供统一 SSE 事件接口。
- [ ] 定义公共事件：`run_started`。
- [ ] 定义公共事件：`status`。
- [ ] 定义公共事件：`plan_summary`。
- [ ] 定义公共事件：`retrieval`。
- [ ] 定义公共事件：`tool_started` 和 `tool_finished`。
- [ ] 定义公共事件：`evidence` 和 `conflict`。
- [ ] 定义公共事件：`token`。
- [ ] 定义公共事件：`citation`。
- [ ] 定义公共事件：`interrupt`。
- [ ] 定义公共事件：`completed` 和 `error`。
- [ ] 所有 API、RAG Schema、Graph 和前端统一使用 `completed`，不得同时使用 `done`。
- [ ] 使用 `messages` 输出答案 Token。
- [ ] 使用 `updates` 获取节点完成状态。
- [ ] 使用 `custom` 输出经过筛选的业务进度。
- [ ] `debug`、完整 `values`、`tasks` 和 `checkpoints` 仅限开发内部。
- [ ] 建立 Event Adapter，将 LangGraph 事件转换成稳定 SSE Schema。
- [ ] 处理客户端断线、任务取消、心跳、超时和背压。
- [ ] 测试事件顺序、重复事件、错误后关闭和断线资源释放。
- [ ] 建立安全测试，确认隐藏推理、Prompt、完整 State 和密钥不会进入 SSE。

验收：前端可以实时看到安全执行进度和答案 Token，但不能看到私有 Chain of Thought。

---

## 17. 阶段 12：短期记忆、滚动摘要与长期记忆

- [ ] 消息原文持久化到 SQL，并按会话顺序读取。
- [ ] 定义短期窗口的消息数或 Token 预算。
- [ ] 超出窗口时触发 AI Rolling Summary。
- [ ] 摘要保存覆盖的 message_id/sequence 范围、Prompt 版本、摘要版本和 `previous_summary_id`。
- [ ] 新摘要基于旧摘要加新增消息滚动更新，不重复总结全部历史。
- [ ] 滚动摘要采用追加新版本，不覆盖或删除旧摘要；旧摘要标记为 superseded。
- [ ] 使用 Conversation 的 `current_summary_id + version` 乐观更新当前摘要指针。
- [ ] 并发摘要更新冲突时放弃过期结果或有限次重新读取后重试。
- [ ] 定义结构化长期记忆类型：用户偏好、项目背景、确认事实、重要实体。
- [ ] 长期记忆只写入稳定、明确且有来源的信息。
- [ ] AI 推断不得自动当作用户事实。
- [ ] 为长期记忆定义稳定 `memory_key`、类型、主体、谓词、值和来源消息。
- [ ] 长期记忆逻辑上按 `memory_key` Upsert，物理上追加不可变 MemoryRevision。
- [ ] 被更新的旧记忆标记为 superseded；被用户否定的记忆标记为 retracted，不直接删除历史。
- [ ] 无法确认新旧事实是否互相替代时保留冲突候选并请求用户确认。
- [ ] 建立长期记忆去重、更新、过期、撤回和隐私删除策略。
- [ ] 建立长期记忆向量检索和相关性门槛。
- [ ] Thinking 加载短期窗口、滚动摘要和相关长期记忆。
- [ ] Fast 默认只加载受限短期上下文，不因记忆显著增加延迟。
- [ ] 测试摘要漂移、错误事实传播和删除后的不可召回。

验收：长对话不超上下文预算；摘要和记忆变更可追溯、可回滚；并发更新不会覆盖新版本；长期记忆不会静默保存模型猜测。

---

## 18. 阶段 13：Web Search、工具、MCP 与 Human-in-the-loop

- [ ] 定义统一 Tool Schema：名称、用途、输入、输出、超时、重试、成本和权限。
- [ ] 实现 Web Search Tool 接口。
- [ ] Web Search 只在本地证据不足、需要最新信息或用户明确要求时触发。
- [ ] Web Search 最多一次，并保存查询和来源信息。
- [ ] 将网页结果转换成统一 Data Source/Evidence，不能直接注入 Prompt。
- [ ] 建立 SQL/Data API Tool 接口。
- [ ] Tool 只提供 Orchestrator 调用包装，Web/SQL/Data API 的实际 Client 统一由 DataSourceAdapter 实现，禁止重复连接器。
- [ ] 保留 MCP Client 接口，但不默认自动发现和调用所有 MCP 工具。
- [ ] 工具选择由单 Orchestrator 决定，并受白名单和调用次数限制。
- [ ] 建立 Human Interrupt：关键歧义、关键来源冲突、高风险操作或缺少必要选择时暂停。
- [ ] Human Interrupt 通过 Checkpoint 恢复。
- [ ] 测试工具超时、无结果、恶意输出、重复调用和用户拒绝继续。

验收：工具调用可追踪、可限制、可恢复；未经处理的外部内容不能控制系统行为。

---

## 19. 阶段 14：模糊问题、无答案与数据冲突

- [ ] 定义“明确、可自动改写、必须询问用户”三类歧义。
- [ ] 可自动改写的问题先转成明确 Query，再进行 Embedding。
- [ ] 改写结果保留原问题、主体、时间、地点和单位。
- [ ] 无法安全确定意图时使用 Human Interrupt，不猜测关键条件。
- [ ] 证据不足时明确拒答或说明缺少什么信息。
- [ ] 识别数值、时间、定义、版本和来源冲突。
- [ ] 定义来源优先级、发布时间和版本判断规则。
- [ ] 冲突可解释时并列展示差异和来源。
- [ ] 关键冲突无法解决时请求用户选择，不擅自合并数字。
- [ ] 所有冲突结论必须带 Citation。
- [ ] 将模糊、无答案和冲突样本加入回归集。

验收：系统不会把改写后的猜测当原问题，也不会用单一来源掩盖真实冲突。

---

## 20. 阶段 15：Evidence、Citation 与生成安全

- [ ] 定义统一 Evidence Schema。
- [ ] 合并本地、Web、SQL 和工具来源。
- [ ] 实现去重、相关性、可信度、时效性和覆盖度计算。
- [ ] 建立 Evidence Sufficiency 判断。
- [ ] 定义 Claim 与 Evidence 的映射。
- [ ] 每条关键事实、数字和日期都应有可定位引用。
- [ ] 引用必须来自实际进入生成上下文的证据。
- [ ] 检查 Citation Precision、Recall、Correctness 和 Coverage。
- [ ] 检查 Unsupported Claim Rate。
- [ ] Prompt Injection 文本作为不可信证据处理，不得升级为系统指令。
- [ ] 对无证据答案和虚假引用建立回归测试。

验收：随机抽样可以从答案 Claim 追溯到原始文档、网页或数据记录。

---

## 21. 阶段 16：Fast A/B 与 RAG 评估

- [ ] 冻结 Fast A/B 的 corpus、queries、qrels、reference answers 和 required evidence。
- [ ] 未经双人独立审核和必要裁决的 Qrels 标记为 weak/provisional。
- [ ] 计算 Hit@1/3/5/10。
- [ ] 计算 Recall@1/3/5/10。
- [ ] 计算 Precision@1/3/5/10。
- [ ] 计算 MRR@K。
- [ ] 计算 nDCG@K，并保留 0/1/2 分级相关性。
- [ ] 计算 MAP@K。
- [ ] 分别保存 Retrieval 原始结果和 Rerank 后结果用于诊断。
- [ ] 分别报告 BM25 Recall/MRR、Dense Exact KNN Recall/MRR 和 RRF Recall/MRR/nDCG。
- [ ] 测量 HNSW 相对 Exact KNN 的 Recall 损失，并记录 HNSW 参数。
- [ ] 报告 Candidate Recall、RRF 后 Recall 和 Cross-Encoder 后 Top-N Recall/MRR/nDCG。
- [ ] 按地名、年份、数字、法规名称、中英文混合、同义表达和无答案问题分组。
- [ ] 记录 BM25 构建时间、索引大小、加载时间、p50/p95/p99 查询延迟和零召回率。
- [ ] 计算 Answer Accuracy、F1、数字/日期/实体准确率。
- [ ] 计算 Faithfulness、Answer Relevance 和 Context Relevance。
- [ ] 计算 Citation Precision、Recall 和 Correctness。
- [ ] 计算无答案识别准确率和任务 Success Rate。
- [ ] LLM-as-judge 记录模型、Prompt 和版本，并用人工样本校准。
- [ ] 记录 Query Embedding、Retrieval、Rerank、TTFT 和总延迟。
- [ ] 延迟报告样本数、mean、p50、p90、p95、p99。
- [ ] 冷缓存和热缓存分别测试。
- [ ] 记录错误、超时、重试、空召回、吞吐量和成本。
- [ ] 建立预先登记的升级门槛，不能看到最终结果后改门槛。
- [ ] 预先指定主要指标、非劣化指标、最小有意义提升和允许退化边界，并报告逐 Query 配对置信区间。
- [ ] 报告不得混入 Thinking 请求或 Thinking Graph 延迟。

验收：同一测试输入可重复生成 A/B 报告；所有结果可追溯到模型、索引、数据和配置版本。

---

## 22. 阶段 17：Service 层完整业务流程

- [ ] ChatService 创建 ChatRun 并保存用户消息。
- [ ] ChatService 为同一 Conversation 分配单调递增的消息和 Run `sequence_number`。
- [ ] 新请求先持久化为 `pending`，通过 Conversation Dispatcher 原子争抢执行槽。
- [ ] 同一 Conversation 同时最多执行一个 Fast 或 Thinking Run；同一用户的不同 Conversation 允许并行。
- [ ] Dispatcher 使用 SQL 状态作为真值，只用短 Redis Lock 或短 SQL Lock 完成选取和状态切换。
- [ ] 当前 Run 完成、失败或取消后，自动选择最早的 Pending Run。
- [ ] Thinking 进入 Human Interrupt 时变成 `waiting_user` 并阻塞本 Conversation 的后续 Run。
- [ ] 用户恢复原 Run 或明确取消后，才继续调度该 Conversation 队列。
- [ ] ChatService 根据用户 mode 调用 Fast 或 Thinking。
- [ ] 当前 Fast 在线服务只读取配置选定的 Fast A Active Snapshot，不调用 A/B Assignment；未来只有获批 Live A/B 才可以增加 Assignment，Thinking 始终绕过它。
- [ ] ChatService 保存最终答案、Citation、状态和延迟。
- [ ] ChatService 处理取消、超时和失败状态。
- [ ] ConversationService 管理会话和消息历史。
- [ ] DocumentService 登记文档并提交 Celery 任务。
- [ ] TaskService 返回任务状态和失败原因。
- [ ] MemoryService 协调 SQL 记忆与 RAG Memory 策略。
- [ ] EvaluationService 创建 Fast A/B 离线运行和读取报告。
- [ ] Service 不依赖 FastAPI Request/Response。
- [ ] 使用事务避免只保存用户消息却丢失运行状态等不一致。
- [ ] 所有 Service 更新可变记录时传入 expected_version，并处理有限次乐观锁冲突。
- [ ] 使用唯一 client_message_id 处理客户端重试，禁止重复生成同一条用户消息和 Run。
- [ ] 为每个 Service 编写单元测试和必要的数据库集成测试。

验收：CLI 或测试代码无需启动 FastAPI 也能调用 Service 完成业务用例。

---

## 23. 阶段 18：完整 API 与最小前端

- [ ] Chat API 支持显式 `fast | thinking` 模式。
- [ ] Fast API 返回实际 A/B Variant 仅用于内部日志或受控调试，不误导普通用户。
- [ ] Thinking API 不包含 Variant 字段。
- [ ] SSE 前端显示连接状态、执行进度、答案 Token 和引用。
- [ ] 请求入队时 SSE 返回 `queued` 状态和可用的队列位置，不把排队伪装成模型运行。
- [ ] API 支持查询 Pending/Running/Waiting/Completed 状态。
- [ ] API 提供取消 Pending 或当前活动 Run 的明确操作。
- [ ] 前端将 Planner 输出显示为简短计划摘要，不显示隐藏推理。
- [ ] 前端支持 Human Interrupt 的提问、用户选择和继续。
- [ ] 前端支持停止生成和断线重连。
- [ ] 前端显示来源、来源时间和冲突说明。
- [ ] 前端错误信息区分可重试、需要用户操作和不可恢复错误。
- [ ] 编写 Fast、Thinking、断线、重连、Interrupt 的端到端测试。

验收：用户能完成真实的多轮 Fast/Thinking 对话，并在 Thinking 中看到安全的实时执行轨迹。

---

## 24. 阶段 19：可靠性、观测与回归

- [ ] 为每次请求贯穿 request_id、conversation_id、run_id、thread_id。
- [ ] 记录各 RAG 阶段耗时，不记录原始私有 Chain of Thought。
- [ ] 记录模型、Prompt、索引和测试集版本。
- [ ] 建立错误率、超时率、空召回率和工具失败率指标。
- [ ] 建立 Fast A/B 分组与指标仪表数据。
- [ ] 建立 Thinking Node 路径、循环次数、Web/Tool 调用次数指标。
- [ ] 建立日志脱敏规则。
- [ ] 建立数据库、Redis、Celery、模型和外部工具健康检查。
- [ ] 建立 Bad Case 分类和固定回归集。
- [ ] 每次修复 Bad Case 先添加回归测试，不扩展无界探索。
- [ ] 建立同一 Conversation 并发提交测试，验证只产生一个活动 Run，其余按 sequence_number Pending。
- [ ] 全局并发 SSE、模型容量和连接池压力测试当前 **DEFERRED**。
- [ ] 建立故障测试：数据库、Redis、Worker、模型、Web Search 分别不可用。

验收：故障可定位、请求可追踪、敏感数据不泄露、关键回归可自动发现。

---

## 25. 最终整体验收

只有以下全部完成后，项目才可以称为完整 Chatbot：

- [ ] Fast A、Fast B 和 Thinking 的生成式 LLM 均通过配置登记的 DeepSeek API Provider 调用，且不存在静默 LLM 回退。
- [ ] Fast A 可以使用 Semantic Chunking + OpenAI Embedding Dense KNN + 对应 BM25 + RRF + Baseline Cross-Encoder 完成检索、生成和引用。
- [ ] Fast B 可以使用训练 Chunking + 训练 Embedding Dense KNN + 对应 BM25 + RRF + 训练 Cross-Encoder 完成同样流程。
- [ ] Thinking 只使用训练版 BM25/Dense IndexSnapshot 和训练 Cross-Encoder。
- [ ] Exact KNN、HNSW、BM25、RRF 和 Rerank 的配置、版本、延迟及质量指标均可追溯。
- [ ] Fast A/B 使用同一冻结测试集并生成完整比较报告。
- [ ] Thinking 全程只使用训练版模型和索引。
- [ ] Thinking Planner/Orchestrator 有界运行，不出现无限 Rewrite 或工具循环。
- [ ] PostgreSQL Checkpoint 可以恢复 Thinking 和 Human Interrupt。
- [ ] SSE/`astream` 可以实时输出安全进度、答案 Token 和 Citation。
- [ ] 系统不向用户暴露隐藏 Chain of Thought。
- [ ] 短期窗口、滚动摘要和结构化长期记忆可用。
- [ ] 所有可变记录通过整数 `version` 乐观锁避免并发静默覆盖，时间戳只用于审计。
- [ ] 原始消息和摘要/记忆历史版本可追溯；滚动摘要追加版本，长期记忆采用逻辑 Upsert + 物理 Revision。
- [ ] 同一 Conversation 请求严格串行，不同 Conversation 可以并行。
- [ ] Human Interrupt 会阻塞本 Conversation 后续 Run，恢复或取消后继续队列。
- [ ] Redis 只持有短期调度/去重锁，SQL 保存最终队列和一致性状态。
- [ ] Web Search、数据接口、工具和 MCP 接口受限、可追踪、可超时。
- [ ] 模糊问题、无答案和数据冲突有明确处理路径。
- [ ] Redis、Celery、SQL 的职责边界符合目录说明。
- [ ] Retrieval、Generation、Citation、Latency 和 Success Rate 指标可重复计算。
- [ ] Train/Dev/Test 无来源和精确文本泄漏。
- [ ] 弱监督测试集未被错误描述为黄金测试集。
- [ ] 单元、集成、端到端、回归和负载测试达到项目约定门槛。
- [ ] 根 README、目录 README、API 文档、训练说明和运维说明完整。
- [ ] 从空环境按照文档能够安装、迁移数据库、启动依赖、运行 API/Worker、构建索引并完成一次 Fast 和 Thinking 对话。

以下能力不属于当前版本完成条件，只保留接口，需用户以后明确授权再实施：入口限流、安全响应 Header、全局请求/SSE/模型容量槽和跨 Conversation 的容量压力测试。同会话串行 Dispatcher、短协调锁和乐观锁已恢复为当前完成条件。

最终验收记录：

```text
日期：
Git Commit：
数据集版本：
Fast A 索引版本：
Fast B 索引版本：
Thinking 模型/索引版本：
测试命令：
评估报告：
已知限制：
验收人：
```
