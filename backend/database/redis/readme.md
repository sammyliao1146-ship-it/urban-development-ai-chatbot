# Redis

本目录当前负责 Redis 连接、缓存、Fast A/B 稳定分组、短期取消状态和 Celery Broker。

## 当前职责与延期能力

Redis 限流和全局容量令牌当前状态为 **DEFERRED / interface-only**。本阶段不实现 Redis 限流 Key、Token Bucket、全局模型 Semaphore 或 SSE 容量令牌。

Conversation Dispatcher 使用 Redis 分布式短锁减少同一会话的重复调度竞争。短锁必须具有唯一 owner token、TTL、原子获取、仅 owner 原子释放和有限等待；锁只覆盖“选择 Pending Run 并提交 SQL 状态切换”的短临界区。

当前与未来 Adapter 契约：

- `RateLimitStore`：未来保存限流窗口和原子计数；
- `CoordinationLock`：用于索引构建、摘要调度和去重等短期协调；
- `ConversationDispatchLease`：用于争抢会话调度权；
- `ConcurrencyLease`：未来如确有跨进程容量控制需求时使用。

Redis 锁不是最终一致性保障；获取锁后仍必须依赖 SQL 事务和唯一约束。不得用 Redis 锁覆盖完整 LLM、SSE 或 LangGraph 执行周期。限流和全局容量接口默认未启用，不能制造已经具备相应保护的假象。
