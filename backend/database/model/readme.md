# Model

本目录定义 SQLAlchemy ORM 表，不放 Pydantic API Schema 或 PyTorch 模型。

公共字段规则：

- 所有表包含数据库 UTC `created_at`；
- 可变表包含数据库 UTC `updated_at` 和从 1 开始的整数 `version`；
- 软删除数据可以增加 `deleted_at`；
- 不可变事件和制品表只追加，不原地覆盖。

Conversation 保存当前摘要指针和版本；Message 使用 `client_message_id` 幂等键以及会话内 `sequence_number`；ChatRun 使用 `pending / running / waiting_user / completed / failed / cancelled` 状态。

数据库必须保证同一 `conversation_id` 同时最多存在一个 `running` 或 `waiting_user` ChatRun。多个 Pending Run 按 `sequence_number` 排队；活动 Run 完成、失败或取消后再调度下一条。

RollingSummary 每次新增版本并关联旧摘要；LongTermMemory 保存逻辑事实，MemoryRevision 保存不可变历史。索引、模型和评估制品通过版本或 Snapshot 新增，不覆盖旧制品。

IndexSnapshot 需要记录 environment、snapshot_family、Corpus、Chunker、Embedding、向量维度、距离函数、BM25 实现/参数、Analyzer、RRF、Reranker、HNSW 参数、文件路径、文件哈希、状态和 version。状态采用 `building / validating / ready / active / retired / failed`。

数据库唯一约束保证每个 `(environment, snapshot_family)` 最多一个 Active Snapshot。ChatRun 保存实际使用的 `snapshot_id`，保证运行、引用和评估可复现。Snapshot 激活事务和验证规则在 Retrieval 文档中定义。

LangGraph PostgreSQL Checkpointer 使用独立 Schema 和自身迁移管理，不在本目录复制其内部表结构或建立业务 ORM Model。ChatRun 只保存用于业务关联和审计的 `run_id`、Graph 版本、状态及必要的恢复错误；业务代码不得直接修改 Checkpointer 内部表。
