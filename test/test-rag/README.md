# Test RAG

本目录用于建立 Fast 模式专用的 A/B Test，不承载生产聊天接口，也不放训练代码。Thinking 模式不参加本测试。第一阶段的“A/B”特指离线成对 Benchmark，不等同于真实用户流量实验。

测试同时运行两条可比较的流水线：

1. **A：Baseline Fast RAG**：Semantic Chunking → BM25 + OpenAI Embedding KNN → RRF → Baseline Cross-Encoder。
2. **B：Trained Fast RAG**：训练 Chunking → BM25 + 训练 Embedding KNN → RRF → 训练 Cross-Encoder。

这是严格的两组 A/B Test，不做组件消融。两条流水线必须使用同一份冻结语料、查询、相关性标注、答案参考和评估器。生成模型、Prompt、硬件、并发、预热次数和测试轮数保持一致；Chunking、Embedding、Retrieval 和 Rerank 作为各自 RAG 的完整方案参与比较。

## 目录

```text
test/test-rag/
├── README.md                 # 总体目标、比较规则和执行阶段
├── baseline/README.md        # 通用 LangChain 基线 RAG 规范
├── trained/README.md         # 训练版 RAG 加载与比较规范
├── shared-test-set/README.md # 两条 RAG 共用的冻结测试集契约
├── metrics/README.md         # 统一指标、统计方法和通过条件
└── reports/README.md         # 评估结果与对比报告的保存规范
```

这些目录目前只描述将来应该放入什么内容，不包含可执行代码、密钥、真实测试数据或生成报告。

## A/B Test 定义

只比较以下两组：

| 测试组 | RAG 流程 | 用途 |
| --- | --- | --- |
| A：Baseline Fast RAG | Semantic Chunking → BM25 + OpenAI Embedding KNN → RRF → Baseline Cross-Encoder | Fast 通用基线 |
| B：Trained Fast RAG | 训练 Chunking → BM25 + 训练 Embedding KNN → RRF → 训练 Cross-Encoder | 验证 Fast 训练版整体是否优于基线 |

本测试只回答“Fast 训练版 RAG 整体是否比 Fast Baseline RAG 更好”，不判断提升来自 Chunking、Embedding、Retrieval 还是 Rerank。报告不得把整体差异归因到某个单独组件。

Thinking 模式完全排除在本 A/B Test 之外，只使用训练版模型和索引。

## 测试流程

```text
冻结测试语料与查询
        │
        ├── A：Baseline Fast RAG ─────┐
        └── B：Trained Fast RAG ──────┤
                                 ▼
                      相同查询、Qrels 和评估器
                                 ▼
               检索质量 + 响应速度 + 答案质量报告
```

建议按以下顺序逐步实现：

1. 冻结并审计共享测试集。
2. 完成 Baseline Fast RAG 的索引和检索测试。
3. 接入训练产物，生成独立索引快照。
4. 先比较纯检索指标，不调用答案生成模型。
5. 检索结果通过后，再比较端到端答案指标。
6. 对 A、B 两组重复运行，输出延迟分位数和逐查询成对差异。
7. 最后形成是否升级训练模型的结论。

## A/B 阶段边界

1. **Offline paired benchmark（当前计划）**：每个 Query 同时运行 A、B，保存成对指标和延迟；不接触真实用户流量。
2. **Shadow（未来可选）**：用户只收到 A 或已选生产版本，另一版本后台运行并受采样、成本和隐私限制。
3. **Live traffic A/B（当前不做）**：需要稳定身份、实验治理、退出条件和线上安全能力，必须由用户以后明确授权。

训练层和正式测试集仍处于 deferred 状态时，只保留上述契约，不得生成伪造 A/B 结论。

## 必须固定的条件

- 同一批测试文档、测试问题和相关性判断；
- 测试文档不能出现在训练集或开发集中；
- 相同的最终召回数量和最终上下文容量；
- 相同的生成模型、Prompt、温度和最大输出长度；
- 相同硬件、并发、预热策略和缓存策略；
- A、B 使用独立索引，索引清单必须记录模型与参数版本；
- A、B 按同一查询顺序执行，并保存单样本结果；
- 弱监督标注必须明确标成 weak，不得称作 golden test。

## 与项目其他目录的边界

- `rag/`：存放可复用的 Chunking、Embedding、Retrieval、Rerank 和 Graph 能力。
- `train/`：训练模型并输出带版本的模型产物与训练报告。
- `dataset/`：保存数据及其来源、许可、切分和审核信息。
- `test/test-rag/`：只负责冻结测试输入、编排对比、计算指标和形成报告。
- `backend/`：以后只调用已经选定的生产 RAG，不负责定义离线评估标准。

## 不应放入本目录的内容

- OpenAI API Key 或其他密钥；
- 模型训练代码和训练数据构建逻辑；
- 生产 FastAPI Router、Redis 或 Celery 实现；
- 手工修改的生成结果；
- 未记录来源和版本的临时语料；
- 使用测试标签调参后又在同一测试集上报告的结果。
