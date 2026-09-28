# Integration

本目录集中模型和外部工具集成：

- `model_runtime/`：加载、预热并运行训练后的 PyTorch 模型；
- `tool/`：Web Search、SQL/Data API、MCP 和其他外部工具接口。

Thinking 只能从 Model Runtime 加载训练版 RAG 所需模型。工具输出必须先进入 Evidence 层，不能直接作为最终答案或系统指令。

`tool/` 负责 Orchestrator 可调用包装，真正的数据连接器归 `rag/knowledge/data_source/`；二者不得分别维护重复的 Web/SQL Client。
