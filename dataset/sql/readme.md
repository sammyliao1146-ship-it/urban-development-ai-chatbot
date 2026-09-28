# SQL Dataset

本目录保存以后供城市数据问答、SQL/Data API Tool 和评估使用的结构化数据集契约，不保存应用业务数据库的 ORM、Migration 或生产凭据。

可以包含：数据字典、只读 Schema/视图说明、来源查询模板、字段与单位、时间和空间粒度、许可、刷新周期、抽取质量、审核状态、版本 Manifest 和测试 Fixture。

每个数据集至少记录 dataset_id、version、source、license、schema_hash、time_range、geographic_scope、primary_key、字段单位、空值规则、更新时间和内容哈希。模型生成的 SQL 不得直接修改来源；在线工具只允许白名单只读查询或受控参数化 Data API。

训练/开发/测试结构化样本仍需按来源或明确语义组隔离，并保留 provenance。应用 PostgreSQL 表结构和迁移属于 `backend/database/`，不要放在这里。
