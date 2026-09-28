# RAG Schema

本目录定义 RAG 内部各组件之间稳定的数据契约。

可以包含：

- Original Query、Effective Query、Input Assessment、Question Type、Route 和 Planner Plan；
- Chunk、Retrieved Document、Rerank Result；
- Evidence、Evidence Grade、Conflict、Claim、Grounding Result 和 Citation；
- Fast/Thinking 请求与结果；
- LangGraph State、Node Result 和 Interrupt；
- 流式执行事件统一为 queued、run_started、status、plan_summary、retrieval、tool_started、tool_finished、evidence、conflict、token、citation、interrupt、completed、error；
- 记忆读取和写入对象；
- 模型运行输入输出。

Schema 只描述数据结构和校验规则，不实现检索、生成或后端业务流程。后端 API Schema 与 RAG 内部 Schema 应通过明确转换隔离，避免前端协议直接绑死 LangChain/LangGraph 对象。
