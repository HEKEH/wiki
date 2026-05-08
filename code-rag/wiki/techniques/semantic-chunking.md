---
title: Semantic Chunking
date: 2026-05-05
tags: [chunking, semantic-aware, retrieval-quality]
sources: [How-to-Chunk-Code-for-RAG.md, Effective-Chunking-Strategies-for-RAG.md]
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

- [[techniques/recursive-character-text-splitter]] — LangChain 的递归分隔方案，低成本实现语义感知（内容无关）
- [[techniques/character-text-splitter]] — 按指定分隔符切分，适合[[concepts/content-dependent-splitting|内容依赖]]场景
- AST 解析 — 基于语法树切分，更精确但实现复杂
- Embedding 相似度 — 在语义转折点切分，计算成本高

## 实验证据：行级分块 ≈ 语法感知分块

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 在 Long Code Arena 代码补全任务上发现，简单的行级切分与 LangChain 语法感知递归切分（匹配平均长度）性能一致，且行级切分略有优势。

**原因**：代码补全更依赖识别语义相似的代码片段，而非保留层级结构中的父子关系。AST 切分保住了语法边界，但对"找到相似代码片段"这一检索目标帮助有限。

**启示**：语法感知切分并非必须 — 简单的行级切分已能保留足够的代码连贯性，同时实现更简单、语言无关。这与 [[concepts/content-dependent-splitting|内容依赖切分]] 的主张形成有趣对比：在代码补全场景下，内容无关的行级切分已经足够。

## 相关

- [[techniques/code-chunking]] — 代码分块的实践方法
- [[concepts/content-dependent-splitting]] — 内容依赖切分是语义分块的系统性方法
- [[techniques/chunk-overlap]] — 内容无关切分的补偿手段
- [[concepts/retrieval-quality]] — 分块质量直接影响检索效果
- [[sources/src-practical-code-rag-at-scale]] — 行级 ≈ 语法感知的实验证据
