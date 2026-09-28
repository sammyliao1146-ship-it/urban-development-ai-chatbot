# Task

本目录集中管理后端异步任务，目前使用 `celery/`。

Celery 负责文档解析、Chunking、Embedding、索引构建、Fast RAG 离线 A/B Test、批量评估和报告生成。实时 Fast/Thinking 聊天请求不经过 Celery。

