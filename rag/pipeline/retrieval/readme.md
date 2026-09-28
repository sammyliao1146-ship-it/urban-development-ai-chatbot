# Retrieval

本目录统一实现 Sparse、Dense 和 Hybrid Retrieval。项目固定架构为：

```text
BM25 Top-K ───────┐
                  ├→ RRF → 去重/来源限制 → Candidate Set
Dense KNN Top-K ──┘
```

## Sparse Retrieval

- 使用版本锁定的 `bm25s` 构建真正 BM25；
- PostgreSQL `ts_rank/ts_rank_cd` 可以用于其他全文检索实验，但不得标记为 BM25；
- BM25 索引是可重建的派生制品，SQL 中的 Chunk 和来源信息才是事实源；
- Fast A 基于 Semantic Chunk 构建独立 BM25 Snapshot；
- Fast B 与 Thinking 基于训练 Chunk 构建训练版 BM25 Snapshot；
- Thinking 使用 BM25 不属于通用模型回退，因为 BM25 是基于训练 Chunk 的确定性词法召回。

BM25 Manifest 必须记录库版本、方法、k1、b、Analyzer 版本、Corpus Hash、Chunker 版本、Chunk 数和稳定 `doc_index → chunk_id` 映射。

## Analyzer

索引与查询必须使用完全相同的 Analyzer：

- Unicode 规范化；
- 语言识别；
- 英文大小写、可配置 stemming 和 stopwords；
- 中文分词和领域词典；
- 保留地名、年份、数字、法规编号、项目名称和缩写；
- 混合中英文文本采用同一版本规则。

Analyzer 版本不匹配时拒绝加载索引。

## Dense Retrieval

- 存储使用 PostgreSQL + pgvector；
- 离线正确性基准使用 Exact KNN；
- 在线生产使用 HNSW ANN；
- Fast A 使用 OpenAI Embedding 和独立向量索引；
- Fast B 与 Thinking 使用训练 Embedding 和训练版向量索引；
- 必须记录向量维度、归一化、距离函数、HNSW M、ef_construction 和 ef_search。

每次版本升级都要测量 HNSW 相对 Exact KNN 的 Recall 损失。

## RRF

BM25 Score 和向量相似度量纲不同，禁止直接相加。两路结果通过 Reciprocal Rank Fusion 合并：

```text
rrf_score(document) = Σ 1 / (rank_constant + rank)
```

`rank_constant`、Sparse Top-K、Dense Top-K 和融合窗口必须配置化，只能在开发集上确定。

RRF 后依次执行：

1. 相同 Chunk 去重；
2. 高重叠文本去重；
3. 必要的相邻 Chunk 合并；
4. 每个 Document/Source 的候选数量限制；
5. 形成供 Cross-Encoder 使用的 Candidate Set。

去重、重叠处理和相邻 Chunk 合并不得丢失叶子级来源。合并 Candidate 必须保留全部 `leaf_chunk_ids` 和 `source_spans`，不能生成一个无法回到原文的临时 Chunk ID。

## Metadata Filter

过滤字段包括 knowledge base、language、city、document type、published time、review status 和 IndexSnapshot。

稳定粗粒度过滤可使用预计算 Bitset 或分区索引；动态过滤可以扩大候选后再过滤并补足。强制过滤不能只依靠偶然的召回后筛选，否则会造成空结果和隐藏 Recall 损失。

## IndexSnapshot

IndexSnapshot 是一套不可拆分、不可原地修改的 RAG 检索版本，至少绑定 Corpus、Chunk、Chunk 映射、BM25、Dense Index、Embedding、Analyzer、RRF 和 Reranker。Model Registry 管单个模型；IndexSnapshot 管这些模型与语料/索引的可运行组合。

Snapshot 按家族隔离：

- `fast_a`：Semantic Chunking + OpenAI Embedding + Baseline Reranker；
- `trained`：训练 Chunking + 训练 Embedding + 训练 Reranker，供 Fast B 和 Thinking 使用。

Celery 按以下流程构建和发布：

```text
building
→ validating
→ ready
→ SQL 原子切换 active
→ 旧版本 retired
```

激活必须满足：

1. 验证 Manifest、文件哈希、模型/Tokenizer、向量维度、Analyzer、Chunk 映射和最小召回测试；
2. 使用短 Redis CoordinationLock 减少同一 Snapshot 家族重复激活，但不能把 Redis 当最终真值；
3. 在一个 SQL 事务内锁定对应 Snapshot 家族，确认候选仍为 `ready` 且 `version` 未变化；
4. 将旧 `active` 改为 `retired`，将新 Snapshot 改为 `active`，两步必须同事务提交；
5. 数据库唯一约束保证每个环境、每个 Snapshot 家族最多一个 `active`；
6. 事务失败时保持旧 Active 不变，不能出现半切换。

每个 ChatRun 在开始执行时解析并保存确定的 `snapshot_id`。该 Run 的 BM25、Dense、Chunk 映射和 Reranker 全程使用此 ID，即使之后发生新版本激活也不能中途切换。回滚通过把已验证的 retired Snapshot 重新原子激活完成。

在线请求只读取 `active` Snapshot；离线验证可以显式读取 `ready` Snapshot。在线路径不得自行选择“最新目录”，也不得把 Fast A 和 trained 家族的组件混合。

统一 Retrieval Result 至少包含：

```text
chunk_id
document_id
document_version
source_id
text
content_hash
start_offset
end_offset
leaf_chunk_ids[]
source_spans[]
retrieval_hits[]:
  - retriever: bm25 | dense
    raw_score
    rank
rrf_score
hybrid_rank
snapshot_id
metadata
```

同一 Chunk 同时命中 BM25 和 Dense 时，`retrieval_hits` 必须保留两条记录；不能用单个 `raw_score/original_rank` 覆盖另一条召回路径。原始分数只用于诊断，RRF 只使用 Rank。
