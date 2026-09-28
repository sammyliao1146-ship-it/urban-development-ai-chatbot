# Urban Development AI Chatbot

本仓库规划一个模块化单体 Chatbot：FastAPI + PostgreSQL/pgvector + Redis + Celery，RAG 使用 LangChain/LangGraph，训练模型由 PyTorch/Sentence Transformers/Chonky 路线产生。

当前状态是 **architecture-first / documentation stage**：目录和接口已经规划，但 `main.py`、Router、依赖清单、训练函数、训练制品和正式测试集尚未实现，项目目前不可直接启动。

## 在线模式

- **Fast A**：Semantic Chunking + bm25s BM25 + OpenAI Embedding KNN + RRF + Baseline Cross-Encoder；第一条可实现路径。
- **Fast B**：训练 Chunker + BM25 + 训练 Embedding KNN + RRF + 训练 Cross-Encoder；没有 Active trained Snapshot 时关闭。
- **Thinking**：有界 LangGraph Planner/Orchestrator，只使用训练版 Snapshot；没有训练制品时关闭。

Fast 负责低延迟固定流程；Thinking 负责输入检查、题型路由、有界检索纠错、Web/工具、冲突处理、Human Interrupt 和最终答案 Grounding。Thinking 的 LangGraph Checkpoint 统一持久化到 PostgreSQL，使用 `run_id` 隔离每次执行。项目不采用微服务，不实现多 Agent、无限反思或无限工具探索。

## 目录

```text
backend/   FastAPI、SQL/Redis、Celery 和应用 Service
rag/       在线模式、Pipeline、LangGraph、Knowledge 和 Integration
dataset/   数据集、来源、许可、切分与审核资料
train/     当前只保留未来训练接口和上游版本约束
test/      Fast A/B 离线成对评估及其他测试入口
frontend/  SSE、Fast/Thinking 和 Human Interrupt 的前端契约
```

详细实施顺序、禁止事项和验收清单见 `agent.md`。正式编码前先完成阶段 0：固定 Python/依赖管理、配置、日志、测试命令和启动说明。
