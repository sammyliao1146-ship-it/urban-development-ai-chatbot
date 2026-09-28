# Pipeline

本目录集中 RAG 数据处理和查询流水线组件：

```text
Chunking
  ├→ bm25s BM25 Index
  └→ Embedding → pgvector Dense Index

Query
  ├→ BM25 Top-K
  └→ Dense Exact KNN/HNSW Top-K
          ↓
       RRF Fusion
          ↓
  去重与来源数量限制
          ↓
 Cross-Encoder Rerank
          ↓
 Evidence → Generation
```

组件应提供统一输入输出，以便 Fast A/B 的两条链路和 Thinking 的训练版链路复用。这里不放 FastAPI、SQL Repository 或 LangGraph 业务路由。

BM25、Dense 和 Reranker 的版本必须绑定到同一个不可变 IndexSnapshot。Fast A、Fast B/Thinking 不得跨 Snapshot 混用 Chunk、向量或模型。
