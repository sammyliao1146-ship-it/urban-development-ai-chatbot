# Baseline Fast RAG

本目录描述 Fast 模式 A 组：Semantic Chunking → bm25s BM25 + OpenAI Embedding pgvector KNN → RRF → Baseline Cross-Encoder → Top-N Evidence。

将来实现时必须记录：

- Semantic Chunking 的实现版本、断点算法、阈值、长度范围、Tokenizer 和后备切分规则；
- OpenAI Embedding 的完整模型标识、维度、归一化、批量、超时、重试、输入 Token 和成本；
- bm25s 版本、k1、b、Analyzer、Sparse Top-K 和索引哈希；
- pgvector 距离函数、Exact KNN/HNSW 参数、Dense Top-K 和元数据过滤；
- RRF rank constant、融合窗口、去重和来源限制；
- Baseline Cross-Encoder 名称、版本、候选数、保留数和并列排名规则；
- 冷/热缓存、硬件、并发、预热次数和失败请求。

API Key 只能通过环境变量或密钥管理系统提供，不得写进测试文件、报告或 Git。

本基线是 Fast A 组的完整方案。Fast B 组可以使用训练流程定义的完整检索与 Rerank 方案，但两组必须使用相同的生成模型、Prompt、测试集和评估器。A/B 结果只用于判断整套 Fast RAG 的差异，不用于推断某个单独组件的贡献。

本方案不能被 Thinking 模式调用。

将来每次运行应生成 Baseline 索引清单、逐查询召回与排序、Rerank 前后结果、分阶段延迟、检索指标、可选的答案与引用指标，以及错误样本清单。
