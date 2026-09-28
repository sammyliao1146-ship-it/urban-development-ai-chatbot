# Metrics

Fast A、B 两组使用同一套指标。报告必须保留逐查询结果、总体汇总、分组结果和失败样本，不能只给一个综合分数。Thinking 不参与这些 A/B 指标。

## Retrieval

至少计算：

- Hit@K；
- Recall@K；
- Precision@K；
- MRR@K；
- nDCG@K；
- MAP@K。

建议报告 `K = 1、3、5、10`，并保留 Retrieval 原始结果和 Rerank 后结果用于诊断。正式 A/B 结论只比较两套完整 RAG 的最终结果，不计算单组件贡献。还应按可回答性、问题类型和难度分组。

内部诊断必须分别保存：

- BM25 Recall、MRR、nDCG、零召回率和查询延迟；
- Dense Exact KNN Recall、MRR 和查询延迟；
- HNSW 相对 Exact KNN 的 Recall 损失；
- RRF 后 Candidate Recall、MRR 和 nDCG；
- Cross-Encoder 后最终 Top-K Precision、Recall、MRR 和 nDCG。

## Chunking

比较训练后的 Chunker 时，记录：

- 边界 Precision、Recall、F1；
- TP、FP、FN、TN 和 Average Precision；
- Chunk Token 长度均值及分位数；
- 过长、过短和空 Chunk 比例；
- 必要证据完整保留率和证据切断率；
- 每文档 Chunk 数、吞吐量和处理延迟。

纯二元段落边界分类不使用 BIO 指标。

## End-to-end answer

纯检索测试通过后，再使用相同生成模型和 Prompt 比较：

- Exact Match 或任务适配 Accuracy；
- Token/字段 F1；
- 数字、日期和实体准确率；
- 必要事实覆盖率；
- Faithfulness；
- Answer Relevance 与 Context Relevance；
- Citation Precision、Recall 和 Correctness；
- Unsupported Claim Rate；
- 无答案识别准确率；
- 人工任务成功率和成对偏好。

LLM-as-judge 只能作为一个评估信号。必须记录评审模型、Prompt 和版本，并以人工样本校准。

## Latency and resources

在线阶段分别记录 BM25、Query Embedding、Exact KNN/HNSW、RRF、Cross-Encoder Rerank、Context 构建、首 Token、完整生成和端到端延迟。

离线阶段分别记录规范化、Chunking、Embedding、索引写入时间，文档/Chunk 吞吐量，API 请求与 Token 用量，以及 CPU、GPU 和峰值内存。

每个延迟指标至少报告样本数、平均值、p50、p90、p95、p99。冷缓存和热缓存分开；失败和重试不得静默排除。限流当前未实现；未来启用后，限流结果也必须单独报告而不能静默排除。

## Promotion rule

Fast B 组训练版成为候选版本前，应同时满足预先登记的质量门槛和延迟预算。至少保证 Recall、MRR、nDCG、Faithfulness、引用正确性和无答案识别不退化，p95 延迟、错误率、超时率和空召回率不超过约定范围。

评估计划必须预先指定一个主要指标、若干非劣化指标、最小有意义提升和允许退化边界。A、B 使用逐 Query 成对结果，报告配对 Bootstrap 或等价方法的置信区间；不能只依据总体均值决定升级。

具体阈值应在建立 Baseline 后预先登记，再运行最终测试；不能看到最终结果后临时修改通过标准。弱标签结果必须标为 provisional，不得用于最终升级结论。
