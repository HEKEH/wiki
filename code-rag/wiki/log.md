# Code RAG — Activity Log

Append-only chronological record of wiki activity.

## [2026-05-05] ingest | How to Chunk Code for RAG

- 首次入库，来源：The Neural Base
- 创建 source 页面：[[sources/src-how-to-chunk-code-for-rag]]
- 创建 technique 页面：[[techniques/code-chunking]], [[techniques/recursive-character-text-splitter]], [[techniques/semantic-chunking]]
- 创建 concept 页面：[[concepts/token-limits]], [[concepts/retrieval-quality]]

## [2026-05-05] ingest | Effective Chunking Strategies for RAG

- 来源：Cohere 官方文档
- 创建 source 页面：[[sources/src-effective-chunking-strategies-for-rag]]
- 创建 technique 页面：[[techniques/chunk-overlap]], [[techniques/character-text-splitter]]
- 创建 concept 页面：[[concepts/content-dependent-splitting]]
- 更新页面：[[techniques/code-chunking]], [[techniques/recursive-character-text-splitter]], [[techniques/semantic-chunking]], [[concepts/retrieval-quality]]
- 核心贡献：分块策略三层框架、内容依赖 vs 内容无关切分、overlap 实验证据

## [2026-05-05] query | 分块策略观点对比

- 对比两篇来源文章的共识与分歧
- 创建分析页面：[[analysis/chunking-strategies-comparison]]
- 核心发现：两篇共识在语义边界优先和 RecursiveCharacterTextSplitter；分歧在 overlap 权重、内容依赖切分的必要性、讨论范围

## [2026-05-07] ingest | Practical Code RAG at Scale

- 来源：JetBrains Research, arXiv 2510.20609
- 创建 source 页面：[[sources/src-practical-code-rag-at-scale]]
- 创建 technique 页面：[[techniques/bm25-retrieval]], [[techniques/sparse-vs-dense-retrieval]]
- 创建 concept 页面：[[concepts/pl-pl-vs-nl-pl-retrieval]]
- 更新页面：[[techniques/code-chunking]]（chunk-size vs context-window）, [[techniques/semantic-chunking]]（行级 ≈ 语法感知）, [[concepts/retrieval-quality]]（延迟-质量权衡、任务模态）, [[concepts/token-limits]]（上下文预算决定最优分块）
- 核心贡献：任务模态驱动检索策略（PL→PL 用 BM25，NL→PL 用 dense）、chunk 大小与上下文窗口对齐、行级分块 ≈ 语法感知、延迟差异高达 200×

## [2026-05-08] query | BM25 实现方式与 Dense Encoder 操作流程

- 补充 BM25 评分公式、实现流程、第三方库到 [[techniques/bm25-retrieval]]
- 补充 Dense Encoder 完整实现流程（离线索引 + 在线查询）、模型选择、向量数据库、关键注意事项到 [[techniques/sparse-vs-dense-retrieval]]
- 补充 Hybrid 混合检索架构（Dense 主检索 + BM25 补充）到 [[techniques/sparse-vs-dense-retrieval]]
- 更新 [[home]] 核心洞见 #1：NL→PL 主导项目推荐 Hybrid 混合检索

## [2026-05-08] lint | 检索口径与来源字段

- [[concepts/retrieval-quality]]：E5-large 按 PL→PL / NL→PL 分写延迟；BPE 倍数与 [[techniques/bm25-retrieval]] 对齐；dense 延迟表述指向实验表；frontmatter 增加 practical-code-rag-at-scale.md
- [[techniques/sparse-vs-dense-retrieval]]：Dense 注意事项与 Hybrid 流程表述澄清（后续勘误：NL→PL 最强 dense 商用名为 voyage-code-3，见同日 lint 条目）
- [[techniques/bm25-retrieval]]、[[concepts/pl-pl-vs-nl-pl-retrieval]]、[[sources/src-practical-code-rag-at-scale]]：sources 改为 practical-code-rag-at-scale.md（与 raw 副本文件名一致）

## [2026-05-08] lint | 模型命名：voyage-code-3

- NL→PL dense 最强基准的商用模型为 **voyage-code-3**（Voyage AI）。wiki 中误写的 Voyager-3-Code 已统改为 voyage-code-3；实验表脚注保留论文表记 `Voyager-3-code` 以便对照 PDF
