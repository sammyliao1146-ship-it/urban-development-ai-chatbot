# Rerank

本目录固定使用 Cross-Encoder 对 Hybrid Retrieval 的候选集进行精排。

```text
BM25 + Dense KNN
→ RRF Candidate Top-N
→ Cross-Encoder(query, chunk)
→ Final Evidence Top-K
```

Cross-Encoder 输入只包含 Query、Chunk 正文、标题和允许的来源字段，不输入会直接泄露目标标签的元数据。

模式规则：

| 模式 | Reranker |
| --- | --- |
| Fast A | Model Registry 中登记的 Baseline Cross-Encoder |
| Fast B | 训练后的领域 Cross-Encoder |
| Thinking | 与训练版 Snapshot 匹配的训练 Cross-Encoder |

Thinking 和 Fast B 在训练 Reranker 不可用或版本不兼容时必须明确失败，不得静默回退 Fast A 的 Baseline Reranker。

统一结果至少包含：

```text
chunk_id
hybrid_rank
reranker_model_id
rerank_score
rerank_rank
latency_ms
```

实现要求：

- 候选窗口和最终 Top-K 配置化；
- 支持 Batch 推理、最大长度和实际 Tokenizer 截断检查；
- 排名并列时使用确定性 tie-break；
- 记录模型、Tokenizer、Snapshot 和 Prompt/Query 版本；
- Rerank 前后结果都要保存用于诊断；
- 不把 Reranker 当生成模型或事实判断器。

评估至少报告 Candidate Recall、最终 Precision/Recall、MRR、nDCG、p50/p95/p99 延迟、吞吐量和内存。
