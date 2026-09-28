# Chunking

本目录定义文档切分统一接口。Chunking 只在文档导入、索引构建和离线评估中运行，不在在线 Query 请求中重新切文档。

统一输入至少包含：`document_id`、`document_version`、规范化正文、标题/章节结构、语言、来源与内容哈希。统一输出 Chunk 至少包含：

```text
chunk_id
document_id
document_version
chunker_id
chunker_version
text
start_offset
end_offset
token_count
heading_path
content_hash
source_metadata
```

`chunk_id` 必须由稳定文档版本、原文范围和 Chunker 版本确定性生成。任何清洗、合并或重叠窗口都必须保留原文 offset，供 Evidence 和 Citation 回溯。

模式规则：

- Fast A：Semantic Chunking，记录实现版本、断点算法、阈值、最小/最大 Token、Tokenizer 和确定性后备切分；
- Fast B/Thinking：加载 Registry 登记的 Chonky ModernBERT Chunker；不可用时明确失败，不回退 Fast A；
- 自动章节、布局、语义和边界标签属于弱监督，除非 Manifest 明确标为人工验证。

长度必须使用该路径的实际 Tokenizer 验证。过长 Chunk 采用记录在 Manifest 中的确定性后备规则；空 Chunk、重复 Chunk、无来源范围 Chunk 和跨文档合并一律拒绝进入 Snapshot。

评估至少记录边界 Precision/Recall/F1、TP/FP/FN/TN、Average Precision、Chunk Token 长度分布、空/过长/过短比例、必要证据完整率、证据切断率、吞吐量和延迟。二元边界任务不使用 BIO 指标。

Chunking Adapter 不负责 Embedding、索引写入、在线路由或模型训练；训练逻辑属于 `train/`，索引发布属于 Celery 与 Retrieval Snapshot。
