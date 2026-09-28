# Database

本目录集中所有数据库和持久化相关内容，避免 Model、Repository、Redis 分散在 `backend/` 根目录。

```text
backend/database/
├── model/       # SQLAlchemy ORM 表结构
├── repository/  # 面向 Service 的 SQL 数据访问
├── redis/       # Redis 连接、缓存键和短期状态
└── readme.md    # Engine、Session、事务、迁移、Checkpoint 和整体边界
```

本目录还负责 SQL Engine、Session Factory、连接池、事务、迁移、数据库健康检查，以及 PostgreSQL、pgvector 和 LangGraph Checkpointer 的连接配置。

`model/` 只定义表；`repository/` 只进行持久化访问；Redis 当前只用于缓存、短期状态和 Celery Broker。业务编排仍在 `backend/service/`。

## 并发与版本规则

- 所有持久化记录使用数据库生成的 UTC `created_at`；
- 所有可变记录包含数据库生成的 `updated_at` 和从 1 开始的整数 `version`；
- 更新使用 `id + expected_version` compare-and-swap，并在成功时原子 `version + 1`；
- Message 使用 `client_message_id` 幂等键和 `conversation_id + sequence_number` 唯一约束；
- 同一 Conversation 允许多个 Pending Run，但同时只能有一个 `running` 或 `waiting_user` Run；
- PostgreSQL 部分唯一索引或等价数据库约束必须作为最终防线；
- Dispatcher 在短事务内选择最早 Pending Run 并切换为 Running；
- 不同 Conversation 可以并行。

SQL 是队列、版本和 ChatRun 状态的最终真值。Redis 只用于缩小竞争窗口和减少重复调度，不能代替事务、唯一约束或乐观锁，也不能持有覆盖完整 LLM/SSE/LangGraph 执行过程的长锁。

## PostgreSQL Checkpointer

LangGraph Checkpoint 固定使用 PostgreSQL 持久化。开发、集成测试和生产环境不得使用仅存在于进程内的 MemorySaver 作为正式 Checkpointer；纯 Graph 单元测试可以注入 Test Double，但不能据此宣称恢复能力已经验收。

- Checkpointer 使用独立 Schema，例如 `langgraph_checkpoint`，与业务表所在 Schema 分离；
- Checkpointer 表由选定并锁定版本的 LangGraph PostgreSQL Checkpointer 迁移机制管理，不在 `model/` 重复建立 SQLAlchemy ORM 映射，也不由普通 Repository 直接修改；
- 可以复用同一个 PostgreSQL 集群，但使用独立连接池或明确的池配额、获取超时和 Statement Timeout，避免 Checkpoint 写入耗尽业务查询连接；
- `thread_id=run_id`，`checkpoint_ns` 记录兼容的 `graph_version/State Schema` 版本；`conversation_id` 只保存在 State 和业务表中，不作为 Checkpoint thread；
- Checkpoint 只保存恢复所需的受控 State。密钥、完整 Prompt、无上限网页正文、大型二进制和模型内部 Chain of Thought 不得写入；大对象保存稳定引用、哈希和必要摘要；
- Checkpoint 提交与 ChatRun 业务事务属于两个明确事务边界，即使位于同一 PostgreSQL 集群也不得假设自动原子提交；通过幂等键、状态条件、心跳/恢复扫描和可审计错误进行协调；
- `waiting_user` 和仍可恢复的 Run 不得被清理。已完成、失败、取消和过期 Checkpoint 按配置保留期由有界 Celery 清理任务分批删除；
- 应用启动只执行兼容性和健康检查。Schema 初始化或迁移作为显式部署步骤执行，不在每个请求或每次 Graph 调用时自动运行；
- Checkpointer 不可用时，Thinking 启动检查失败或当前 Run 明确失败，不得静默切换内存 Checkpointer。Fast 模式不依赖 Graph Checkpoint，可按其自身依赖健康状态处理。
