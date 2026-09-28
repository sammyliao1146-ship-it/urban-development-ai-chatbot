# Handler

本目录是 FastAPI 的 HTTP/SSE 入口层。Handler 应保持轻薄，只负责协议适配，然后调用 `backend/service/` 完成业务。

应该按功能组织：

- Chat：Fast/Thinking 请求、SSE 流和取消；
- Conversation：创建、查询、列表和历史消息；
- Document：上传、登记和处理状态；
- Task：Celery 任务状态和错误；
- Evaluation：Fast A/B 离线运行和报告读取；
- Health：存活、就绪和版本信息。

Handler 的职责：

1. 接收 Path、Query、Header 和 Body；
2. 使用 `backend/fastapi/schema/` 校验请求；
3. 通过 Dependency 获取 Service 和请求级资源；
4. 调用 Service；
5. 将 Service 结果转换成 HTTP Response 或 SSE Event；
6. 将已知异常交给统一 Exception Handler；
7. 在客户端断线时取消或清理对应流式资源。

SSE Handler 只向前端发送稳定公共事件，例如：

```text
queued
run_started
status
plan_summary
retrieval
tool_started
tool_finished
evidence
conflict
token
citation
interrupt
completed
error
```

不得发送隐藏 Chain of Thought、原始 Prompt、完整 LangGraph State、Checkpoint、密钥或未脱敏工具参数。

Handler 不应该：

- 直接执行 SQL 或操作 ORM；
- 直接读写 Redis；
- 拼装 Prompt、调用 Embedding/Retrieval/Rerank；
- 定义 LangGraph Node 或循环；
- 实现 A/B 分组算法、记忆更新或索引切换；
- 把 FastAPI Request/Response 对象传入 Service。

`router.py` 可以作为聚合各功能 Router 的入口，但业务代码应继续位于 Service 和 RAG。
