---
title: Semantic Chunking
date: 2026-05-05
tags: [chunking, semantic-aware, retrieval-quality]
sources: [How-to-Chunk-Code-for-RAG.md]
---

# Semantic Chunking

在切分文本时保持语义完整性的分块策略，而非简单按固定长度切分。

## 与固定长度切分的区别

- **固定长度**：按 N 个字符/token 切，不考虑内容边界 → 容易切断句子、函数、逻辑单元
- **语义分块**：优先在逻辑边界（段落、函数、类）处切分 → 每个 chunk 保持自洽

## 代码场景的语义边界

- 函数/方法定义（`def` / `class`）
- 类的完整定义（方法不被拆散）
- 代码块之间的空行（`\n\n`）
- import 区域

## 实现方式

- [[techniques/recursive-character-text-splitter]] — LangChain 的递归分隔方案，低成本实现语义感知
- AST 解析 — 基于语法树切分，更精确但实现复杂
- Embedding 相似度 — 在语义转折点切分，计算成本高

## 相关

- [[techniques/code-chunking]] — 代码分块的实践方法
- [[concepts/retrieval-quality]] — 分块质量直接影响检索效果
