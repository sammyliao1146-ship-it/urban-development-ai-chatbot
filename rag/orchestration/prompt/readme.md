# Prompt

本目录集中管理 RAG 使用的 Prompt 模板和版本说明。

应该按用途组织：

- Input Guard、错误前提检查、问题澄清和 Query Rewrite；
- Question Router、Planner 与 Orchestrator；
- Evidence Grading、时效性判断和冲突判断；
- 内部答案草稿、Claim-Evidence Grounding、最终答案生成与拒答；
- 滚动摘要和长期记忆抽取；
- Citation 检查和评估 Judge。

每个 Prompt 应有稳定 ID、版本、输入变量、适用模型和变更记录。Prompt 不应散落在 Router、Repository 或数据库模型中，也不能包含 API Key、连接信息或真实用户隐私数据。
