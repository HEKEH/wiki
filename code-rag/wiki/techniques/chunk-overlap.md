---
title: Chunk Overlap
date: 2026-05-05
tags: [chunking, overlap, retrieval-quality]
sources: [Effective-Chunking-Strategies-for-RAG.md]
---

# Chunk Overlap

相邻 chunk 之间保留一定重叠区的策略，用于补偿切分时丢失的上下文。

## 原理

在内容无关切分中，语义单元（句子、发言人、函数）可能被切断到两个相邻 chunk 的边界处。Overlap 在相邻 chunk 之间保留一段重复文本，使被切断的信息有机会在下一个 chunk 中完整出现。

## 实验证据

Cohere 的转录稿实验（[[sources/src-effective-chunking-strategies-for-rag]]），使用 [[techniques/recursive-character-text-splitter|RecursiveCharacterTextSplitter]]：

- chunk_size=500, overlap=0 → 发言人名字被切到另一个 chunk，模型无法识别说话者
- chunk_size=500, overlap=100 → 同一信息被 overlap 区捕获，答案正确

## 权衡

| 方面 | 优势 | 劣势 |
|------|------|------|
| 检索完整性 | 边界信息不丢失 | — |
| 冗余 | — | 重复内容增加存储和 embedding 计算 |
| 检索效率 | — | 同一信息可能被多个 chunk 命中，降低 top-k 的有效覆盖面 |

## 配置建议

- `chunk_overlap` 通常设为 `chunk_size` 的 10%~20%
- 内容依赖切分（[[concepts/content-dependent-splitting]]）可从根源避免语义断裂，减少对 overlap 的依赖
- 代码场景中，overlap 可防止函数签名/返回语句被截断

## 相关

- [[techniques/code-chunking]] — 代码分块实践
- [[concepts/retrieval-quality]] — overlap 影响检索质量
- [[concepts/content-dependent-splitting]] — 更彻底的解决方案
