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
