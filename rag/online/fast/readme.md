# Fast RAG

本目录封装在线低延迟 Fast 模式。Fast 是唯一进行 A/B Test 的在线模式。

固定流程建议为：

```text
问题规范化
→ 可选的轻量歧义处理
→ BM25 + Dense KNN 并行召回
→ RRF
→ Cross-Encoder Rerank
→ Evidence Selection
→ Answer Generation
→ Citation
```

Fast A 使用 Semantic Chunking、对应 BM25、OpenAI Embedding Dense KNN 和 Baseline Cross-Encoder。Fast B 使用训练 Chunking、对应 BM25、训练 Embedding Dense KNN 和训练 Cross-Encoder。两组使用独立 Snapshot，共用冻结测试集、生成模型和统一指标。

训练制品尚不存在时，Fast A 是唯一可启用在线路径，Fast B 返回明确的 capability unavailable，不得使用未登记模型或空实现。Fast B 只有在 Active trained Snapshot 验证通过后才能启用。

Fast 模式复用 `rag/pipeline/` 和 `rag/knowledge/citation/`，但不进入 Planner/Orchestrator 循环，不调用 Web Search，也不做多轮反思。

这里放 A/B 路由、Pipeline 组装、Fast 模式配置和统一入口，不放底层模型实现、FastAPI Router 或数据库访问。
