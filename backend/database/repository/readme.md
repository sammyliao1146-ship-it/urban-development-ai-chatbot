# Repository

本目录是 Service 与持久化实现之间的数据访问边界。

应该按业务对象组织会话、消息、文档、Chunk、记忆、实验、任务和评估结果的读取与写入，并隐藏 ORM 和具体 SQL 细节。

Repository 只负责：

- 查询、写入、更新和分页；
- 数据库对象与后端 Schema 的必要转换；
- 明确的事务参与方式；
- 批量读写和锁定策略。

## 乐观锁与队列契约

对可变记录执行 compare-and-swap：

```text
WHERE id = target_id AND version = expected_version
SET ..., version = version + 1
```

影响行数为零时返回统一并发冲突，交由 Service 有限次重读重试或重新入队。不得使用 `updated_at` 代替整数版本，也不得静默采用最后写入者覆盖。

Repository 负责 Message 幂等键和会话序号唯一，并提供按 Conversation 原子选择最早 Pending ChatRun 的操作。选择和状态切换必须在同一短事务中完成；数据库唯一约束保证同一 Conversation 只有一个活动 Run。

不应该包含 FastAPI Request/Response、Prompt、LLM 调用、LangGraph 路由或跨业务流程编排。

项目不再单独保留 CRUD 目录。简单表操作和面向业务聚合的数据访问统一由 Repository 对 Service 暴露，避免两套数据访问层重复。
