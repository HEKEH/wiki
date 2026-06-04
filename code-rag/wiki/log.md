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

## [2026-05-12] ingest | Repoformer: Selective Retrieval for Repository-Level Code Completion

- 来源：Wu et al., arXiv 2403.10059v2
- 创建 source 页面：[[sources/src-repoformer-selective-retrieval]]
- 创建 system 页面：[[systems/repoformer]]
- 创建 concept 页面：[[concepts/selective-rag]]
- 更新页面：[[concepts/retrieval-quality]]（检索决策维度）、[[techniques/code-chunking]]（Repoformer 索引分块配置）、[[techniques/sparse-vs-dense-retrieval]]（选择性检索与稀疏检索）、[[concepts/pl-pl-vs-nl-pl-retrieval]]（PL→PL 检索必要性低）、[[techniques/bm25-retrieval]]（Jaccard + 选择性策略）
- 核心贡献：检索不总是有帮助（仅 20% 提升、20% 有害）、选择性 RAG 框架（self-assessment 最优）、模型自评估避免不必要检索、最高 70% 推理加速
- 勘误：A4 消融结果初始误写为 UT 25→26，实为 ES 从 ~53 退化到 ~25，已修正

## [2026-06-02] lint | frontmatter 修复

- 为 `sources/src-repoformer-selective-retrieval.md` 的含冒号标题加引号（避免 YAML 解析失败）

## [2026-06-04] ingest | Improve RAG Performance Using Cohere Rerank

- 来源：AWS Machine Learning Blog（Shashi Raina @AWS、Pradeep Prabhakaran @Cohere），2024-09-16
- 创建 source 页面：[[sources/src-improve-rag-performance-cohere-rerank]]
- 创建 technique 页面：[[techniques/reranking]]（两阶段检索、cross-encoder vs bi-encoder、Cohere Rerank、RRF vs 重排）
- 更新页面：[[techniques/sparse-vs-dense-retrieval]]（Hybrid 流程加入重排环节、RRF vs 重排说明、相关链接）、[[concepts/retrieval-quality]]（新增"重排/两阶段检索"质量维度 + frontmatter source）、[[concepts/pl-pl-vs-nl-pl-retrieval]]（重排受益于 NL→PL 的相关链接）、[[home]]（核心洞见 #11、两条开放问题）
- 核心贡献：两阶段检索范式（粗召回 + 精排）、cross-encoder 联合编码精度高于 bi-encoder 但无法预计算、重排即插即用修复第一阶段排序质量、NL→PL 场景收益最大

## [2026-06-04] lint | Cohere Rerank ingest 自查

- [[sources/src-improve-rag-performance-cohere-rerank]]：标注源文笔误 `786` 应为 `768`（BERT-base 维度）；补全遗漏的 Rerank 3 Nimble 低延迟变体
- [[techniques/reranking]]：维度改为正确的 768–1536；"其他重排路径"（开源 cross-encoder / RankGPT / FlashRank）加引用块标注为非本源补充知识，RankGPT 标为待 ingest 候选（arXiv 2304.09542, EMNLP 2023）
- 结论：cross-encoder/bi-encoder 术语框架、候选规模 50–200 等为符合既有惯例的通用工程知识补充（参见 [2026-05-08] query 先例）；事实性无误
