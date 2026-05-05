---
title: Retrieval Quality
date: 2026-05-05
tags: [retrieval, quality, evaluation]
sources: [How-to-Chunk-Code-for-RAG.md]
---

# Retrieval Quality

RAG 系统中检索结果的相关性和完整性，直接决定生成质量。

## 影响因素

- **分块策略**：[[techniques/semantic-chunking|语义分块]] 优于固定长度切分，破碎 chunk 导致检索结果不可用
- **Chunk 大小**：过大则信息密度低、相关性稀释；过小则缺乏上下文
- **Embedding 质量**：代码 embedding 需要理解语法结构，通用文本模型未必合适
- **Overlap**：合理的 [[techniques/code-chunking|chunk overlap]] 补偿切分丢失的上下文

## 相关

- [[techniques/code-chunking]] — 分块是检索质量的基础
- [[techniques/semantic-chunking]] — 语义感知的分块提升检索质量
