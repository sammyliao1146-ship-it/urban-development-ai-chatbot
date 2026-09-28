# Train：上游依赖与预留接口

本目录当前处于 **interface-only / deferred implementation** 状态。这里只规定未来训练能力的来源、输入输出和制品契约。

当前阶段禁止：

- 创建训练函数或训练脚本；
- 创建 Dataset Builder；
- 生成 train/dev/test 数据；
- 下载模型或执行训练；
- 安装、复制或修改上游仓库；
- 生成 Checkpoint、预测、指标或测试集；
- 为了通过接口而加入假的训练实现。

只有用户以后明确授权“开始实现训练”或指定某个训练阶段时，后续 Agent 才能解除对应限制。

## 1. 固定上游来源

### 1.1 Embedding 与 Cross-Encoder

未来 Embedding 和 Cross-Encoder 训练必须基于 Hugging Face 官方 Sentence Transformers GitHub 源码：

```text
repository: https://github.com/huggingface/sentence-transformers.git
reference commit: 75ae64ed5d4c62ca4be3e905c26a7a2cee41e266
```

正式引入时必须：

- 使用 Git Submodule、锁定 Git Revision 或等价可复现方式；
- 在 Manifest 中记录 URL、Commit、License 和本地修改；
- 优先使用上游 SentenceTransformer/CrossEncoder 训练能力；
- 不复制大段上游源码到业务目录；
- 上游接口不满足需求时通过本项目 Adapter 包装，而不是直接改 Vendor 源码；
- 升级 Commit 时创建新训练运行和兼容性报告，不覆盖旧模型。

### 1.2 Chunking

未来 Chunking/边界模型训练必须基于 Chonky GitHub 源码及其 ModernBERT 路线：

```text
repository: https://github.com/mirth/chonky.git
reference commit: 796f75e2d895b05ed3f8002d2bddf87bfd13f1c0
default model family: ModernBERT
default candidate: mirth/chonky_modernbert_base_1
optional larger candidate: mirth/chonky_modernbert_large_1
```

第一版默认以 ModernBERT Base 作为资源基线；只有显存、延迟和评估预算允许时，才将 Large 作为独立模型版本评估。不得在未记录的情况下自动切换模型。

Chonky README 将这两个 ModernBERT 模型标记为非多语言模型。若未来训练数据包含中文或多语言内容，必须先记录语言覆盖限制并单独评估；不能把非多语言 ModernBERT 的英文结果直接外推到中文，也不能静默替换成其他模型家族。

## 2. 未来目录边界

以后解除实现限制时，训练层最多按以下职责组织，不提前创建空代码：

```text
train/
├── agent.md            # 本文件
├── chunking/           # Chonky ModernBERT 训练 Adapter
├── embedding/          # Sentence Transformers Embedding 训练 Adapter
├── reranker/           # Sentence Transformers CrossEncoder 训练 Adapter
├── interface/          # 训练 Job 与 Artifact 契约
└── manifest/           # 上游版本与训练制品 Schema
```

训练实现不得放进 `rag/`。训练后的模型由 `rag/integration/model_runtime/` 加载，训练数据以后由 `dataset/` 提供，运行制品以后写入新的版本目录。

## 3. 预留训练任务接口

以下是数据契约，不是 Python 函数签名。

### TrainingJobSpec

所有未来训练任务共享：

```text
job_id
model_type: chunker | embedding | reranker
upstream_repository
upstream_commit
base_model_id
dataset_reference
output_directory
seed
device
precision
resume_checkpoint
hyperparameters
created_at
```

### TrainingArtifactManifest

所有训练结果必须输出：

```text
artifact_id
model_type
model_id
version
checkpoint_path
tokenizer_id
upstream_repository
upstream_commit
base_model_id
dataset_version
train_split_hash
dev_split_hash
test_split_hash（只有以后存在正式测试集时）
weak_supervision_status
seed
hyperparameters
thresholds
metrics
limitations
created_at
git_commit
```

## 4. Chunker 接口契约

未来 Chonky ModernBERT Adapter 需要支持：

- 接收经过审核的 Dataset Reference，而不是自行扫描任意目录；
- 使用实际 ModernBERT Tokenizer 验证最大长度；
- 输出边界/Token Classification 所需的统一预测；
- 保存最佳 Checkpoint、最后 Checkpoint、Tokenizer 和阈值；
- 支持 Resume，但恢复时校验上游 Commit、模型 ID、数据版本和超参数；
- 生成二元 Precision、Recall、F1、TP/FP/FN/TN 和 Average Precision；
- 将自动边界标签明确标记为弱监督；
- 禁止使用 BIO 指标代替二元边界指标；
- 禁止把直接决定标签的章节/边界元数据传给模型。

## 5. Embedding 接口契约

未来 Sentence Transformers Adapter 需要支持：

- Query、Positive 和显式 Negative；
- SentenceTransformer 模型及其 Tokenizer；
- MNRL 或经过明确批准的 Retrieval Loss；
- 防止共享 Positive 的 Query 在同一 Batch 中成为假负例；
- 保存模型、Tokenizer、Pooling/Normalize 配置和训练报告；
- 使用开发集报告 Recall@K、MRR@K 和 nDCG@K；
- 记录 Batch Sampler、Loss、Learning Rate、Epoch、Seed 和最大长度；
- 禁止使用测试集选择模型、阈值或超参数。

## 6. Reranker 接口契约

未来领域 Cross-Encoder 使用 Sentence Transformers 的 CrossEncoder 路线：

- 输入 Query/Chunk Pair 和明确相关性标签；
- 记录候选由哪个 Retriever/Snapshot 生成；
- 输出可排序相关性分数；
- 保存模型、Tokenizer、最大长度和评分方向；
- 报告 Rerank 前后 Recall、MRR、nDCG、延迟和吞吐量；
- Fast B 与 Thinking 只能加载 Model Registry 中登记的训练 Reranker。

## 7. Dataset 占位契约

当前不创建任何训练集、开发集或测试集，只保留未来 Dataset Reference 应满足的条件：

- Train/Dev/Test 按来源文档或明确语义组隔离；
- 检查 Source、Exact Text、Normalized Text 和 Query 重叠；
- 保存 Source ID、URL、License、抽取质量和审核状态；
- 自动生成标签属于弱监督；
- 未经独立人工审核的测试集不得称作 Golden Test；
- 所有长度使用实际模型 Tokenizer 验证；
- Test 不参与训练、阈值选择、Prompt 调整或模型选择。

`test/test-rag/` 当前也只保留 A/B Test 规范，不代表已经存在正式测试数据。

## 8. Runtime 接口

训练层未来只产生版本化 Artifact；在线层只通过 Model Registry 和 Model Runtime 使用它：

```text
train
→ TrainingArtifactManifest
→ Model Registry
→ rag/integration/model_runtime
→ Fast B / Thinking
```

Runtime 不导入 Trainer，不在服务启动时训练，也不从未登记目录自动寻找“最新”Checkpoint。

## 9. 解除限制前的确认清单

- [ ] 用户明确授权开始实现某个训练方向。
- [ ] 上游仓库 URL、Commit 和 License 已确认。
- [ ] Base Model ID 和语言覆盖已确认。
- [ ] Dataset 来源、拆分、审核状态和许可已确认。
- [ ] 新输出目录、Seed、设备和资源预算已确认。
- [ ] 评估指标与通过门槛已预先登记。
- [ ] 明确是否允许创建开发集和测试集。

以上条件未满足时，只能修改接口文档，不能实现或运行训练。
