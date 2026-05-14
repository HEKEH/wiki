# Home

Top-level synthesis and orientation for this knowledge base.

## About

This wiki is incrementally built and maintained by an LLM agent. Raw sources are immutable; the wiki is the LLM's layer — summaries, concept pages, technique pages, cross-references, and evolving synthesis.

## Quick Links

- [[index]] — catalog of all wiki pages
- [[log]] — chronological activity record

## Current State

This wiki is focused on **Code RAG** — how LLMs retrieve and use codebase context, code search, embedding strategies, and tooling for code-aware AI.

### 核心洞见

1. **检索策略必须匹配任务模态** — PL→PL 用 [[techniques/bm25-retrieval|BM25]]，NL→PL 用 dense encoder；不存在万能配置（[[concepts/pl-pl-vs-nl-pl-retrieval]]、[[techniques/sparse-vs-dense-retrieval]]）；NL→PL 主导的项目推荐 Hybrid 混合检索
2. **Chunk 大小与上下文窗口对齐** — ≤4K tokens 用 32-64 行，≥16K tokens 可用整文件；小上下文需精确聚焦，大上下文可容纳完整代码段（[[techniques/code-chunking]]）
3. **分块是 Code RAG 的基础** — [[techniques/code-chunking]] 的质量直接决定检索和生成效果，语义完整性比切分效率更重要
4. **简单行级分块 ≈ 语法感知分块** — [[techniques/semantic-chunking|语义分块]] 实验证明行级切分与 AST 切分性能一致，实现复杂度无需提高
5. **字符 ≠ Token** — [[concepts/token-limits]] 是最常见的陷阱：`RecursiveCharacterTextSplitter` 的 `chunk_size` 是字符数，代码约 4 字符/token
6. **检索质量受延迟约束** — 不同配置延迟差异高达 200×，BM25+词级是 PL→PL 的最佳质量-延迟平衡点（[[concepts/retrieval-quality]]）
7. **内容依赖切分从根源解决语义断裂** — [[concepts/content-dependent-splitting]] 利用文档结构定制切分规则，优于依赖 overlap 的内容无关切分
8. **Chunk overlap 是双刃剑** — [[techniques/chunk-overlap]] 补偿切分边界的信息丢失，但引入冗余；内容依赖切分可减少对 overlap 的依赖
9. **检索并非总是必要** — 代码补全中仅 ~20% 检索提升性能，~20% 反而有害；[[concepts/selective-rag|选择性 RAG]] 通过模型自评估避免不必要检索，同时提升精度和效率（[[systems/repoformer|Repoformer]]）
10. **自评估是 Selective RAG 的最优决策机制** — 比传统 trial retrieval / trial generation 更低延迟且更准确；模型结合"自知性"和"任务依赖性"两个信号判断（[[concepts/selective-rag]]）

## Open Questions

- 代码分块是否存在比行级切分更优的方案（AST 感知分块在哪些场景下有优势）？
- 代码 embedding 模型（如 voyage-code-3）与通用文本 embedding 在检索质量上的差异有多大？
- Chunk overlap 的最优比例是否存在领域相关的经验值？
- 内容依赖切分的预处理成本与检索质量提升之间的 ROI 如何量化？
- 上下文压缩（去重、摘要）能否改变小上下文模型的最优 chunk 策略？
- 多行补全和代码修复任务是否与单行补全遵循相同的检索规律？
- Selective RAG 的自评估机制能否推广到 NL→PL 场景（如代码问答）？
- 不同仓库的"RAG 友好度"差异如何量化？能否实现个性化选择性检索策略？
- 自监督标签基于词法相似度 (ES)，执行验证 (UT) 是否能进一步提升选择准确性？
