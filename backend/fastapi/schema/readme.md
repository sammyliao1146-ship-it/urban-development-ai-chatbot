# Schema

本目录定义 FastAPI 对外的 Pydantic 请求、响应和 SSE 事件格式，是 HTTP 协议边界。

建议按领域组织：

- ChatRequest、ChatResponse；
- ConversationCreate、ConversationResponse；
- MessageResponse、HistoryResponse；
- DocumentCreate/UploadResponse；
- TaskResponse；
- EvaluationRequest、EvaluationResponse；
- Pagination、ErrorResponse；
- StreamEvent 及各事件 Payload。

Chat 请求至少表达：

```text
conversation_id
client_message_id
mode: fast | thinking
message
expected_version（并发更新时执行乐观锁）
```

流式事件使用稳定的判别字段，例如：

```text
event_type
request_id
conversation_id
run_id
sequence_number
timestamp
payload
```

Thinking 的 Human Interrupt 恢复请求必须携带原 `run_id`；后端从 ChatRun 查询并使用该值作为 LangGraph `thread_id`。客户端不得自定义新的 Checkpoint thread ID。

规则：

- 对外字段有明确类型、必填性、长度和枚举限制；
- 使用 UTC 时间和清晰的序列化格式；
- 错误响应包含稳定错误码、可公开信息和是否可重试；
- Fast 的 A/B Variant 只用于内部日志或受控调试，不默认暴露；
- Thinking Response 不包含 A/B Variant；
- API 版本变化应保持向后兼容或显式升级版本；
- 不直接返回 SQLAlchemy ORM、LangChain Document、LangGraph State 或第三方 SDK 对象。

本目录不定义数据库表；ORM 位于 `backend/database/model/`。RAG 内部 Query、Evidence、Citation、Plan 和 Graph State 位于 `rag/orchestration/schema/`，通过 Service/Adapter 转换为 API Schema。

限流、安全 Header 和全局容量控制当前只保留未来错误码、配置和字段扩展位置，不要求当前 Schema 返回限流配额或容量令牌。

同会话并发控制当前启用。Schema 需要表达 `pending/running/waiting_user/completed/failed/cancelled`、队列序号、乐观锁冲突和可重试状态；不得向客户端暴露 Redis 锁 owner token 或数据库内部锁信息。
