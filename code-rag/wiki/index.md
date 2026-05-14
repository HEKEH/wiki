# Code RAG — Index

Catalog of all wiki pages, organized by category. Updated on every ingest.

## Overview

- [[home]] — 顶层综合与导航，核心洞见和开放问题

## Systems

- [[systems/repoformer]] — 基于 StarCoderBase 的选择性 RAG 代码补全系统；模型自评估是否需要跨文件检索

## Techniques

- [[techniques/code-chunking]] — 将代码切分为逻辑块以适配 RAG 的 token 限制和语义边界；chunk 大小应与上下文窗口对齐
- [[techniques/recursive-character-text-splitter]] — LangChain 的递归分隔文本分块工具（内容无关）
- [[techniques/character-text-splitter]] — LangChain 的按指定分隔符切分工具（内容依赖）
- [[techniques/semantic-chunking]] — 在逻辑边界处切分以保持语义完整性；行级切分 ≈ 语法感知切分
- [[techniques/chunk-overlap]] — 相邻 chunk 之间保留重叠区以补偿切分丢失的上下文
- [[techniques/bm25-retrieval]] — 基于词频的稀疏检索，PL→PL 场景的最优选择
- [[techniques/sparse-vs-dense-retrieval]] — 稀疏 vs 密集检索的任务依赖性对比

## Concepts

- [[concepts/token-limits]] — LLM 上下文窗口的 token 计量与字符换算；上下文预算决定最优 chunk 大小
- [[concepts/retrieval-quality]] — 检索结果的相关性和完整性；受任务模态、计算预算、延迟约束共同影响
- [[concepts/content-dependent-splitting]] — 根据文档结构定制切分规则，与内容无关切分相对
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — PL→PL 与 NL→PL 检索模态的本质差异及策略选择
- [[concepts/selective-rag]] — 选择性检索：系统主动判断是否需要检索，避免不必要的检索开销和噪声

## Sources

- [[sources/src-how-to-chunk-code-for-rag]] — How to Chunk Code for RAG (The Neural Base)
- [[sources/src-effective-chunking-strategies-for-rag]] — Effective Chunking Strategies for RAG (Cohere)
- [[sources/src-practical-code-rag-at-scale]] — Practical Code RAG at Scale (JetBrains Research, arXiv 2510.20609)
- [[sources/src-repoformer-selective-retrieval]] — Repoformer: Selective Retrieval for Repository-Level Code Completion (Wu et al., arXiv 2403.10059)

## Analysis

- [[analysis/chunking-strategies-comparison]] — The Neural Base vs Cohere 分块策略观点对比
