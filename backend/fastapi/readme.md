# FastAPI

本目录集中放所有直接依赖 FastAPI 的内容，避免 Router、Schema、依赖注入和异常处理散落在 `backend/` 根目录。

```text
backend/fastapi/
├── handler/     # API Router、SSE 聊天入口和健康检查入口
├── schema/      # Pydantic 请求、响应和流式事件
├── dependency/  # FastAPI Depends 和请求级资源
├── exception/   # HTTP/SSE 异常映射
└── middleware/  # 日志、请求 ID、耗时、CORS 等中间件
```

本目录只负责 HTTP 协议层。业务流程交给 `backend/service/`，SQL 和 Redis 交给 `backend/database/`，RAG 逻辑交给 `rag/`。

