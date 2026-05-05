# Code RAG — Index

Catalog of all wiki pages, organized by category. Updated on every ingest.

## Overview

- [[home]] — 顶层综合与导航，核心洞见和开放问题

## Systems

<!-- - [[systems/Name]] — one-line summary -->

## Techniques

- [[techniques/code-chunking]] — 将代码切分为逻辑块以适配 RAG 的 token 限制和语义边界
- [[techniques/recursive-character-text-splitter]] — LangChain 的递归分隔文本分块工具（内容无关）
- [[techniques/character-text-splitter]] — LangChain 的按指定分隔符切分工具（内容依赖）
- [[techniques/semantic-chunking]] — 在逻辑边界处切分以保持语义完整性的分块策略
- [[techniques/chunk-overlap]] — 相邻 chunk 之间保留重叠区以补偿切分丢失的上下文

## Concepts

- [[concepts/token-limits]] — LLM 上下文窗口的 token 计量与字符换算
- [[concepts/retrieval-quality]] — 检索结果的相关性和完整性，直接决定生成质量
- [[concepts/content-dependent-splitting]] — 根据文档结构定制切分规则，与内容无关切分相对

## Sources

- [[sources/src-how-to-chunk-code-for-rag]] — How to Chunk Code for RAG (The Neural Base)
- [[sources/src-effective-chunking-strategies-for-rag]] — Effective Chunking Strategies for RAG (Cohere)

## Analysis

- [[analysis/chunking-strategies-comparison]] — The Neural Base vs Cohere 分块策略观点对比
