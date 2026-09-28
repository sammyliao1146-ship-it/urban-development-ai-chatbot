# Orchestration

本目录集中 LangChain 和 LangGraph 编排：

- `chain/`：可复用的小型 LCEL Chain；
- `graph/`：State、Node、Routing、Planner、Orchestrator 和 Checkpoint 使用方式；
- `prompt/`：Prompt 模板和版本；
- `schema/`：RAG 内部数据与流式事件契约。

Fast 模式只组合固定 Chain；Thinking 模式使用有界 LangGraph。这里不保存训练模型文件。

