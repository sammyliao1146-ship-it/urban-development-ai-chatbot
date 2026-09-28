# Dependency

本目录放 FastAPI 的依赖注入定义，用于把基础设施和业务服务安全地提供给 Router。

应该放入：

- SQL Session、Redis Client、Celery Client 的依赖；
- Service、Repository 和 RAG Facade 的构造依赖；
- 请求级配置、事务范围和资源清理；
- FastAPI `Depends` 使用的提供器。

不应该放入业务规则、SQL 查询、RAG 流程或模型推理逻辑。目前项目不做鉴权，因此这里暂不规划用户鉴权依赖，但保留以后扩展接口。

