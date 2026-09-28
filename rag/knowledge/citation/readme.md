# Citation

本目录负责建立最终答案陈述与证据来源之间的可追溯关系。

应该放入：

- Citation 数据结构；
- Chunk、文档、网页和数据接口来源的引用格式；
- 答案 Claim 与 Evidence 的映射；
- 引用去重、编号和排序；
- Citation Precision、Recall、Correctness 和 Coverage 检查；
- 无法定位来源时的失败处理。

Citation 不负责 Retrieval 或答案事实判断。引用必须指向实际使用的证据，不能仅列出检索到但未支持答案的来源。

Citation 必须引用稳定来源定位：document/source ID、document version、leaf chunk ID 或原文 start/end offset，以及内容哈希。合并后的 Evidence 可以对应多个 Citation Span，但不能只引用无法复现的临时合并文本。
