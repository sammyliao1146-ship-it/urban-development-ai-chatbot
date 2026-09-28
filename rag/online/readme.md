# Online RAG

本目录放真正被聊天入口调用的两种在线模式：

- `fast/`：低延迟固定链路，也是唯一参加 A/B Test 的模式；
- `thinking/`：Planner + Orchestrator 的有界图，只使用训练版 RAG。

Fast 与 Thinking 可以复用底层 Pipeline、Knowledge、Orchestration 和 Integration 能力，但两种模式的路由和限制分别在各自目录定义。

当前训练层处于 deferred 状态时，只允许启用 Fast A。Fast B 和 Thinking 必须通过配置关闭；它们只有在 Active trained Snapshot 与全部登记模型可用后才能开放。
