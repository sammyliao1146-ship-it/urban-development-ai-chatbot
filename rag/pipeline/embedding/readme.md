# Embedding

本目录定义 Query/Document Embedding 的统一推理接口和版本契约，不包含训练代码。

统一请求至少包含文本、输入类型 `query|document`、model_id、model_version、Tokenizer、最大长度、前缀策略和 batch 配置。统一结果至少包含向量、维度、归一化状态、模型版本、输入内容哈希、截断信息和延迟。

模式规则：

- Fast A：使用配置中明确登记的 OpenAI Embedding 完整模型 ID；记录维度、归一化、批量、超时、重试、Token 与成本；
- Fast B/Thinking：使用 Registry 登记的训练 Sentence Transformers Embedding；不可用时明确失败，不回退 OpenAI Baseline；
- 文档索引与查询必须使用同一模型版本、维度、归一化、距离函数以及 query/document prefix 约定；
- 不同 Embedding 空间不得共享 pgvector 索引。

缓存键至少包含 model_id、model_version、input_type、prefix_version、normalize、max_length 和规范化文本哈希。查询缓存与文档缓存分开命名；缓存命中也必须返回完整版本元数据。

pgvector Snapshot 必须记录维度、距离函数、是否归一化、Exact KNN/HNSW 参数和 Corpus/Chunk 映射哈希。Runtime 在写入或查询前验证向量维度和 Snapshot Manifest；不兼容时拒绝执行。

Embedding Adapter 不负责 Chunking、RRF、Rerank、模型训练或在线模式选择。API Key 只从配置/密钥系统读取，不写入缓存、日志、Manifest 或测试报告。
