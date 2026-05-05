---
title: Retrieval Quality
date: 2026-05-05
tags: [retrieval, quality, evaluation]
sources: [How-to-Chunk-Code-for-RAG.md, Effective-Chunking-Strategies-for-RAG.md]
---

# Retrieval Quality

RAG 系统中检索结果的相关性和完整性，直接决定生成质量。

## 验证手段

Cohere 的 `co.chat` 提供引用归因功能 — 生成答案时标注每个断言来自哪个 chunk（如 `[1]`），可据此反向验证检索结果是否正确支撑了答案。这对评估分块策略效果尤其有用（[[sources/src-effective-chunking-strategies-for-rag]]）。

## 影响因素

- **分块策略**：[[techniques/semantic-chunking|语义分块]] 优于固定长度切分；[[concepts/content-dependent-splitting|内容依赖切分]] 从根源保证语义完整性
- **Chunk 大小**：过大则信息密度低、相关性稀释；过小则缺乏上下文
- **Embedding 质量**：代码 embedding 需要理解语法结构，通用文本模型未必合适
- **Overlap**：[[techniques/chunk-overlap|Chunk overlap]] 补偿切分丢失的上下文，但引入冗余
- **切分方式与文档的匹配度**：内容无关切分对结构化文档（如转录稿）效果差，Cohere 实验证实无 overlap 时发言人信息丢失导致检索失败

## 相关

- [[techniques/code-chunking]] — 分块是检索质量的基础
- [[techniques/semantic-chunking]] — 语义感知的分块提升检索质量
- [[techniques/chunk-overlap]] — overlap 补偿切分边界的信息丢失
- [[concepts/content-dependent-splitting]] — 内容依赖切分从根源提升检索质量
