# Test

本目录集中测试规范。`test/test-rag/` 负责 Fast A/B 离线成对评估；以后代码测试还应按单元、数据库/Redis/Celery 集成、RAG 组件、LangGraph 路径、API/SSE、并发、Checkpoint 恢复和端到端测试分类。

测试必须使用隔离配置和临时制品，不能连接生产数据库、Redis、外部索引或模型目录。Thinking Graph 可以注入确定性 Test Double，但不得把测试替身登记为生产模型或 Snapshot。
