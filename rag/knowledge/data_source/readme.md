# Data Source

本目录定义 RAG 可以读取的知识和数据来源适配器，并拥有真正访问外部系统的 Client/Connection 实现。

可以包含：

- 本地文档库和向量索引；
- SQL 查询型数据源；
- 城市开放数据 API；
- Web Search；
- 文件、对象存储或其他只读知识源；
- 数据来源健康检查和元数据标准化。

每个数据源必须返回统一来源信息，包括来源 ID、标题、URL 或数据表、时间、许可、版本和可信度信息。这里不负责 Planner 决策；`rag/online/thinking/` 的 Orchestrator 决定何时调用哪个数据源。

统一 `RawSource` 至少包含 source_id、source_type、title、content、locator、published_at、retrieved_at、version、license、content_hash 和原始元数据。Web/SQL/Data API Tool 必须委托这里的 Adapter，不能重复实现客户端。
