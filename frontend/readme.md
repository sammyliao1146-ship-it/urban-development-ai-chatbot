# Frontend Contract

前端只消费后端稳定 HTTP/SSE 协议，不读取 LangGraph 内部 State。界面支持选择 Fast/Thinking、发送消息、显示队列/执行状态、答案、Citation、冲突说明和 Human Interrupt。

SSE 事件统一为：`queued`、`run_started`、`status`、`plan_summary`、`retrieval`、`tool_started`、`tool_finished`、`evidence`、`conflict`、`token`、`citation`、`interrupt`、`completed`、`error`。

同一 Conversation 有 Running 或 WaitingUser Run 时，前端显示状态并禁止重复启动；真正的唯一活动 Run 保证仍由后端 SQL/Service 提供。Human Interrupt 恢复必须提交原 `run_id`。断线重连通过 Run 状态和已保存事件/结果恢复，不能要求 Graph 从 START 重跑。

Thinking 未启用时应显示 capability unavailable，而不是静默切换 Fast。前端只展示安全执行轨迹，不展示隐藏 Chain of Thought、Prompt、Checkpoint、完整 State 或未检查候选答案。
