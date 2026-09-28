# Util

本目录只放无业务状态、可独立测试、被多个 Backend 模块复用的通用小工具。

适合放入：

- UTC 时间与时间格式转换；
- UUID、Hash 和幂等键生成；
- 安全的 JSON 编码辅助；
- 文件名与 MIME 基础校验；
- 分页参数和通用排序辅助；
- 日志字段脱敏；
- 有上限的重试/退避计算；
- 不依赖业务对象的文本、集合和路径辅助函数。

每个工具应满足：

- 输入输出明确；
- 无隐藏全局状态；
- 不在导入时建立连接或加载模型；
- 对相同输入尽量返回确定结果；
- 边界条件有单元测试；
- 名称表达具体用途，避免 `common.py`、`helpers.py` 无限膨胀。

不应该放入：

- Chat、Conversation、Memory 等业务规则；
- SQL 查询、Repository 或事务；
- FastAPI Request/Response；
- Redis/Celery 客户端；
- Prompt、Retrieval、Rerank 或 LangGraph Node；
- PyTorch 模型加载和推理；
- 为逃避正确分层而临时堆放的代码。

如果工具只被一个模块使用，应优先留在该模块内部；只有真正跨模块、无业务语义的能力才进入 Util。
