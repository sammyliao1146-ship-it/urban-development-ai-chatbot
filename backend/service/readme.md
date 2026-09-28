# Service

本目录实现后端应用用例，是 Router 与 Repository、RAG、Redis、Celery 之间的编排层。

应该放入：

- 创建会话、发送消息、读取历史等 Chat Service；
- Fast/Thinking 模式选择和调用；
- 文档上传后的任务创建与状态查询；
- 记忆读写、实验运行和评估任务的应用流程；
- 事务边界、幂等性、超时和业务级错误转换；
- 同步低延迟路径与异步任务路径的衔接。

Service 可以调用 RAG 对外接口，但不应该实现 Chunking、Embedding、Retrieval、LangGraph Node 或具体 SQL，也不直接依赖 FastAPI Request/Response 对象。

## 会话并发规则

- 不按 user_id 全局串行；同一用户的不同 Conversation 可以并行；
- 同一 Conversation 的 Fast/Thinking ChatRun 串行执行；
- 新请求先保存为 Pending，并分配单调递增 `sequence_number`；
- Conversation Dispatcher 使用短 Redis Lock 降低竞争，并在 SQL 短事务内选择最早 Pending Run；
- 数据库唯一约束保证同一 Conversation 同时最多一个 Running 或 WaitingUser Run；
- Human Interrupt 进入 WaitingUser 并继续占用当前 Conversation 执行槽；
- Run 完成、失败或取消后释放槽并调度下一条 Pending Run；
- Service 使用 expected version 更新可变记录，并对乐观锁冲突进行有限次处理；
- 客户端重试使用 `client_message_id` 幂等，不能重复创建消息和 ChatRun。

全局模型/SSE 容量 Semaphore 仍然延期，它与同会话正确性控制是两种不同能力。
