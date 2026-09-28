# Celery

本目录放后端异步任务基础设施和任务入口。

适合交给 Celery 的工作：

- 文档解析、清洗、Chunking、Embedding 和批量写索引；
- 重建索引和切换索引版本；
- Fast RAG 的离线评估与 A/B Test；
- 长对话摘要、批量记忆整理；
- 大规模数据导入、模型预热和报告生成；
- 失败任务重试与定时维护。

应该记录队列划分、超时、重试、幂等键、任务状态和失败处理。实时聊天 Token 流、普通数据库查询和低延迟 Retrieval 不应先经过 Celery。Thinking 模式不参加 A/B Test。

Redis 可以作为 Broker 或短期任务状态存储，但最终业务状态和重要评估结果应写入 SQL。

索引任务同时构建并验证同一 IndexSnapshot 内的 BM25、pgvector Dense Index、稳定 Chunk 映射和 Manifest。验证通过后先进入 `ready`；激活任务再通过短协调锁、SQL 事务、expected version 和数据库唯一约束完成家族内原子切换。在线请求不得修改正在使用的索引，也不得在一个 ChatRun 中途更换 snapshot_id。
