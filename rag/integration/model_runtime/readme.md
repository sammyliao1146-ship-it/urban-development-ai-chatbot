# Model Runtime

本目录统一封装训练模型和外部模型在推理阶段的加载与执行。

应该放入：

- PyTorch Chunker、Embedding、Reranker 等模型加载器；
- Tokenizer、设备、精度和批处理管理；
- 模型预热、健康检查和资源释放；
- 统一的 `load / warmup / run / close` 运行契约；
- 模型 ID、Checkpoint、训练数据和版本清单的解析；
- 加载失败、版本不兼容和回退策略。
- DeepSeek API Provider 的客户端生命周期、健康检查、超时、有限重试、取消和用量记录。

模型文件本身应保存在带版本的训练输出或模型制品目录，不直接复制进这里。本目录不负责训练，也不决定 Fast/Thinking 路由。

Thinking 只允许加载登记过的训练版模型和索引，不提供通用 Baseline 回退。Fast 可以根据 A/B 分组选择通用 Baseline 或训练版模型。

线上生成式 LLM 固定为 DeepSeek API。Generation、Graph 和 Chain 只能依赖统一 Provider 契约，不能直接创建供应商客户端。DeepSeek Provider 不负责 Embedding、BM25 或 Rerank；不可用时明确失败，不得静默切换到其他 LLM。供应商返回的隐藏推理或 reasoning content 不得进入 Graph State、Checkpoint、日志或 SSE。

未来训练来源固定为：

- Chunker：Chonky GitHub 的 ModernBERT 路线；
- Embedding：官方 Sentence Transformers GitHub；
- Reranker：官方 Sentence Transformers CrossEncoder 路线。

当前只保留 Artifact/Registry 接口，不存在训练函数或训练制品。Runtime 不能导入 Trainer、不能在服务启动时训练，也不能扫描目录自动选择“最新”Checkpoint；只能加载 Model Registry 明确登记且 Manifest 完整的版本。

Model Registry 的单模型版本必须由 IndexSnapshot 引用。Runtime 接收明确的 `snapshot_id + model_id`，不能独立挑选“最新”Embedding 或 Reranker。加载后的模型 ID、Tokenizer、维度和文件哈希必须与 Snapshot Manifest 一致。

测试目录可以提供实现相同 Runtime/Retriever Protocol 的确定性 Test Double，用于验证 Graph 分支、Checkpoint 和错误处理。Test Double 必须通过测试依赖注入显式传入，不能出现在生产 Registry、生产配置或 Snapshot Manifest 中。
