# Reports

本目录用于保存将来的 RAG 对比报告规范，当前不包含生成结果。

每次报告需要记录：

- Evaluation Run ID、执行时间和代码版本；
- 共享测试集版本与文件哈希；
- Fast A 组 Baseline 与 Fast B 组 Trained RAG 的完整定义；
- Chunker、Embedding、Retriever、Reranker、生成模型和 Prompt 版本；
- 索引快照 ID、bm25s/Analyzer/k1/b、Sparse/Dense `top_k`、Exact KNN/HNSW、RRF 和 Cross-Encoder 参数；
- 硬件、并发、缓存、预热和重复次数；
- 总体、分组和逐 Query 指标；
- Rerank 前后差异；
- 各阶段延迟、吞吐量和成本；
- 错误、超时、空结果和回退；
- 相对 Baseline 的绝对变化、相对变化和回归样本；
- 是否达到预先登记的升级门槛；
- 主要指标、非劣化边界、逐 Query 成对差异和置信区间；
- 弱监督状态、已知限制和最终结论。

原始逐查询结果与汇总报告应分开保存。每次运行使用新的唯一目录，不覆盖旧结果，也不删除失败运行。报告不得包含 API Key、完整环境变量或其他敏感信息。

报告只比较 A、B 两套完整系统，不提供 Chunking、Embedding、Retrieval 或 Rerank 的消融结论。

报告不得混入 Thinking 请求、Thinking 延迟或 Thinking Graph 指标。

报告必须标明运行类型：`offline_paired`、`shadow` 或 `live_ab`。当前只允许 `offline_paired`；不得把离线 Benchmark 描述成线上用户实验。
