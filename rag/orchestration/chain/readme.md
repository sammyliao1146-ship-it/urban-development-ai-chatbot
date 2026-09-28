# Chain

本目录放可复用的 LangChain/LCEL 小型 Chain，不负责完整业务编排。

可以包含：

- 问题规范化与独立问题生成；
- Query Rewrite；
- 文档相关性和证据充分性评分；
- 上下文格式化；
- 答案生成；
- 引用校验；
- 结构化模型输出解析。

Chain 应具有清晰输入输出 Schema，能够被 Fast Pipeline 或 LangGraph Node 复用。不要在这里访问 FastAPI、直接写数据库或实现无限循环。

