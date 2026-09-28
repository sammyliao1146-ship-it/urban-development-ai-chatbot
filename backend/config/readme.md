# Config

本目录集中管理应用配置的声明、加载、校验和环境差异，不保存真实密钥。

应该放入：

- 应用名称、运行环境、日志等级和调试开关；
- FastAPI Host、Port 和 CORS；
- PostgreSQL、Redis、Celery 的连接配置；
- OpenAI、Web Search、外部数据 API 的配置字段；
- 模型 Registry、PostgreSQL Checkpointer、索引制品根目录和设备配置；
- Fast/Thinking 开关、超时、循环上限和工具调用上限；
- 同会话 ChatRun 串行、Dispatcher 锁 TTL、乐观锁重试和队列配置；
- BM25、Dense KNN、HNSW、RRF、Cross-Encoder 的默认参数；
- 开发、测试、生产环境的配置覆盖规则。

规则：

- 配置从环境变量、配置文件或密钥系统加载；
- 仓库只提交 `.env.example`，不得提交真实 API Key、密码或 Token；
- 启动时校验必填项、类型、范围和互斥配置；
- 测试使用独立配置，不能连接生产数据库、Redis、索引或模型目录；
- 运行期间视配置对象为只读，需要变更时创建新配置版本；
- 模型、Prompt、索引和实验配置必须带版本，便于复现实验。

入口限流、安全 Header 和全局模型/SSE 容量控制当前为 **DEFERRED / interface-only**。配置层只预留 `rate_limit`、`security_headers` 和 `runtime_capacity` 命名空间及启用开关，默认关闭。

同会话并发控制属于当前必须实现的正确性能力：`conversation_concurrency` 默认开启，规定同一 `conversation_id` 同时只能有一个 `running` 或 `waiting_user` ChatRun；不同 Conversation 可以并行。配置可以调整有限重试和锁 TTL，但不能关闭数据库唯一约束。

在训练制品延期期间，`FAST_B_ENABLED=false`、`THINKING_ENABLED=false`。只有存在通过验证且处于 Active 状态的 `trained` Snapshot，并且 Model Runtime 健康检查通过时才能开启；启动校验失败必须保持关闭，不能回退到 Fast A。

LangGraph Checkpoint 固定使用 PostgreSQL，不提供生产或开发环境的内存 Checkpointer 开关。配置至少包含独立的 Checkpointer DSN 或受控复用应用 PostgreSQL DSN、专用 Schema、连接池大小、获取连接超时、Statement Timeout、保留时间、清理批大小和健康检查。真实 DSN 只能来自环境变量或密钥系统。单元测试可以注入 Test Double，但 Checkpoint 恢复、服务重启和 Human Interrupt 集成测试必须连接隔离的 PostgreSQL 测试库，禁止连接开发或生产 Schema。

本目录不创建数据库连接、不加载模型、不执行 Retrieval，也不包含业务判断；这些工作分别属于 Database、Model Runtime、RAG Pipeline 和 Service。
