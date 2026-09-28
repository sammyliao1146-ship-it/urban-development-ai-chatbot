# Generation

本目录负责根据已经选定的证据生成最终答案。

线上生成式 LLM 固定通过 DeepSeek API 调用。模型 ID 由配置明确指定，不在代码或 Prompt 中写死；API Key 只从环境变量或密钥系统注入。Fast A、Fast B 和 Thinking 共用同一 DeepSeek Provider 契约，但可以使用各自经过登记的生成参数和 Prompt 版本。

DeepSeek API 用于答案生成、Query Rewrite、结构化路由、Planner、Evidence Grade、Grounding、摘要和记忆抽取等需要 LLM 的能力。它不替代 OpenAI Embedding、训练 Embedding、BM25、Cross-Encoder Reranker 或本地 PyTorch 模型。

应该包含：

- 上下文组装规则；
- 有证据回答、证据不足拒答和冲突说明策略；
- Fast 与 Thinking 共用的生成入口；
- 流式 Token 输出适配；
- 答案结构、引用占位和生成参数；
- 防止脱离证据回答的约束。

Generation 不负责检索、重排、Web Search、长期记忆写入或 FastAPI SSE 连接。它只接收问题、对话上下文和已整理证据，返回结构化答案结果。

DeepSeek Provider 必须支持超时、有限重试、流式与非流式调用、结构化输出校验、取消、用量和延迟记录。对外事件不得包含 API Key、原始 Prompt、供应商原始响应、隐藏推理或 reasoning content。Provider 不可用或结构化输出连续校验失败时应明确报错，不得静默切换到其他 LLM。

