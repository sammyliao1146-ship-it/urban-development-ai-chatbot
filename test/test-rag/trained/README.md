# Trained Fast RAG

本目录描述 Fast B 组训练版 RAG 的测试契约。训练逻辑不放在这里；这里只加载 `train/` 生成并由模型清单明确标识的完整 RAG 产物与配置。

Fast 训练版作为一个整体参与 A/B Test，固定包含训练 Chunking、对应 bm25s BM25、训练 Embedding pgvector KNN、RRF 和训练 Cross-Encoder。本测试不拆分这些组件，也不做单组件消融。

每个训练产物至少需要提供：

- 唯一模型 ID、版本、模型类型与基础模型；
- Checkpoint 路径和 Tokenizer 标识；
- 输入、输出和批处理约定；
- 训练、开发数据版本以及弱监督状态；
- 随机种子和关键超参数；
- 训练验证指标、适用限制与失败条件；
- 代码提交或等价的可复现标识。

训练版必须复用 `shared-test-set/` 的测试语料、查询、Qrels、参考答案、必要证据、测试顺序和指标规则。生成模型、Prompt、硬件、缓存条件和运行次数必须与 Baseline 相同。

Baseline 与训练版必须建立独立索引，因为 Chunk 边界和向量空间可能不同。

训练版 Snapshot 必须绑定 BM25 Analyzer、训练 Chunker、训练 Embedding、Dense Index、RRF 配置和训练 Cross-Encoder；任何一项版本不匹配都不得激活。

最终报告只得出“B 组完整训练版 RAG 相比 A 组 Baseline RAG 的整体提升或退化”，不得把差异单独归因给 Chunking、Embedding、Retrieval 或 Rerank。

Thinking 也使用训练版 RAG，但不属于本 A/B Test，不生成 A/B 分组，也不调用 Baseline。

测试集不能用于训练、阈值选择或 Prompt 调优。训练、开发、测试应按文档或明确语义组隔离，并检查来源、完整文本、规范化文本和 Query 重叠。
