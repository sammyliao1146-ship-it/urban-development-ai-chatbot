# Tool

本目录封装可以被 LangGraph Orchestrator 调用的工具接口。Tool 是“可调用能力包装”，不重复实现外部数据客户端。

可以包含：

- Web Search Tool；
- SQL/Data API Tool；
- 城市数据计算工具；
- MCP Client 适配接口；
- 以后新增外部服务的统一包装。

每个工具需要声明名称、用途、输入输出 Schema、超时、重试、权限、成本和可公开结果。工具输出必须作为不可信外部数据进入 Evidence 层，不能直接成为系统指令。

职责链固定为：

```text
Orchestrator
→ Tool（调用 Schema、白名单、次数、超时、权限）
→ DataSourceAdapter（真正访问 Web、SQL 或外部 API）
→ RawSource
→ EvidenceNormalizer
→ Evidence
```

Web Search 和 SQL/Data API 的 SDK Client、连接、分页、响应解析与来源元数据只在 `rag/knowledge/data_source/` 实现一份。Tool 只委托对应 DataSourceAdapter，不能建立第二套连接器。

当前只保留单 Orchestrator 的工具调用能力，不扩展为多 Agent 或自动无限工具探索。
