# Shared Test Set

本目录定义 Baseline Fast RAG 与 Trained Fast RAG 共用的只读测试集契约。真实数据以后放入时应带版本和清单，测试运行不得原地修改它。该测试集服务于 Fast A/B，不用于 Thinking 分流。

当前只保留测试集格式和审核规范，不创建、生成或声称已经存在正式测试集。只有用户以后明确授权数据构建与人工审核流程后，才能加入真实 Query、Qrels 和 Reference Answer。

共享测试集需要包含：

- **corpus**：正文、`source_id`、`document_id`、来源、许可、抽取质量和审核状态；
- **queries**：`query_id`、问题文本、问题类型、难度和是否可回答；
- **qrels**：Query 与文档、稳定 passage 或原文范围的相关性等级；
- **reference answers**：参考答案、必须事实、可接受答案和不可回答条件；
- **required evidence**：支撑答案所必需的来源或证据范围。

两个 RAG 的 Chunker 可能生成不同 Chunk ID，因此不要把某一组的临时 Chunk ID 直接当成跨组真值。相关性应优先绑定到稳定文档、原文范围或稳定 passage 标识，再映射到 A、B 两组生成的 Chunk。

建议相关性等级：

- `0`：不相关；
- `1`：相关，但不足以单独回答；
- `2`：高度相关，可直接支持答案。

自动生成、章节推导或其他启发式标签都属于弱监督。它们可以用于流程冒烟和趋势观察，但未经独立人工审核不能称作黄金测试集。两名标注者应独立判断，分歧由第三人裁决。

测试集冻结时必须记录版本、文件哈希、样本数，并检查 source overlap、exact-text overlap 和 query overlap。训练、开发、测试按来源文档或明确语义组隔离；长度使用实际 Tokenizer 检查。
