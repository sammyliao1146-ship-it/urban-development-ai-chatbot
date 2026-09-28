# Middleware

本目录放所有请求级横切逻辑。Middleware 处理每个请求都需要的协议行为，但不实现具体业务。

应该放入：

- Request ID/Trace ID 的生成、透传和响应头；
- 结构化访问日志；
- 请求开始、结束和总耗时统计；
- CORS；
- 未处理异常的最后防线；
- 日志脱敏和敏感 Header 过滤。

## 当前延期能力

入口限流、安全响应 Header 和全局运行容量控制当前状态为 **DEFERRED / interface-only**。本阶段不实现：

- Redis Token Bucket、Sliding Window、计数器或 `429` 限流逻辑；
- CSP、HSTS、Permissions-Policy、X-Frame-Options 等安全 Header 中间件；
- SSE 连接数、模型执行数或请求执行数的 Semaphore/Admission Control；
- 依赖代理 IP、用户 ID 或 API Key 的限流身份识别。

只保留未来插入点和契约名称：`RateLimiter`、`RateLimitPolicy`、`RateLimitDecision`、`SecurityHeaderPolicy`、`RuntimeCapacityController` 和对应配置命名空间。当前运行时不得声称已经启用这些保护；未来实现需要用户再次明确授权。

同会话 ChatRun 串行不在 Middleware 实现。Middleware 只透传 `conversation_id/run_id`；真正的唯一活动 Run 约束由 Service、SQL 事务、数据库唯一约束和 Conversation Dispatcher 保证。

CORS、请求日志和 Request ID 仍属于普通 FastAPI 外壳，不代表已经实现安全 Header 或入口限流。请求 Schema 的字段长度校验也不等同于运行时流量控制。

日志上下文应尽量贯穿：

```text
request_id
conversation_id
run_id
thread_id
mode
path
method
status_code
latency_ms
```

流式请求注意事项：

- 不要预读取或复制完整上传文件和请求体；
- 不要缓存完整 SSE 响应；
- 总耗时在流关闭后记录；
- 客户端断线应向 Handler/Service 传播取消；
- 不记录答案全文、隐藏推理、Prompt、Cookie、Authorization 或 API Key。

Middleware 不负责用户业务、数据库事务、RAG 路由、A/B 分组、模型调用和 Celery 任务定义。当前项目不实现鉴权；以后增加鉴权时应作为独立依赖或专用中间件，而不是散落在 Handler。
