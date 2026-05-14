---
title: PL→PL vs NL→PL Retrieval
date: 2026-05-07
tags: [task-modality, pl-pl, nl-pl, retrieval, code-rag]
sources: [practical-code-rag-at-scale.md, Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# PL→PL vs NL→PL Retrieval

代码 RAG 系统中两种根本不同的检索模态，决定了最优检索策略的选择。

## 术语

- **PL** = Programming Language，编程语言，即代码
- **NL** = Natural Language，自然语言，即人类日常语言（中文、英文等）

## 定义

- **PL→PL**：代码查代码。查询是代码片段，目标是代码片段。典型任务：代码补全。
- **NL→PL**：自然语言查代码。查询是自然语言文本，目标是代码文件。典型任务：Bug 定位、代码搜索、仓库问答。

## 核心差异

| 维度 | PL→PL | NL→PL |
|------|-------|-------|
| 词汇重叠 | 高（同一语言、相同标识符） | 低（自然语言 vs 代码符号） |
| 最优检索 | 稀疏 (BM25) | 密集 (Dense encoder) |
| 原因 | 词汇重叠直接反映相关性 | 需要语义桥接跨模态鸿沟 |
| 延迟要求 | 可用极快方案 (BM25 0.2s) | 需付出更高延迟 (3-19s) |

## 为什么模态决定策略

PL→PL 查询中，用户代码与目标代码共享大量词汇（函数名、类名、变量名），BM25 的词频匹配直接有效。NL→PL 查询中，issue 文本描述问题（如"登录失败"），而代码用不同表达（如 `AuthException`、`validateToken`），词汇重叠极低，必须依赖语义编码建立对应。

## Python 的启示

[[sources/src-practical-code-rag-at-scale|Galimzyanov et al. (2025)]] 发现 Python 仓库中 BM25 在 NL→PL 任务上表现相对更好 (NDCG 0.64 vs E5-large 0.61)。原因：Python 代码自然语言痕迹更丰富（docstring、注释、描述性命名），缩小了 PL 与 NL 的词汇鸿沟。

**推论**：代码库中的自然语言密度（注释、docstring、命名风格）影响稀疏检索在 NL→PL 场景下的有效性。

## PL→PL 场景中检索的必要性

[[sources/src-repoformer-selective-retrieval|Wu et al. (2024)]] 在 PL→PL 代码补全中发现，仅 ~20% 的检索实例提升模型性能，~60% 无影响，~20% 有害。这与 PL→PL 的特性一致：当补全目标可从当前文件的局部上下文推断时（如简单赋值、循环结构），跨文件检索是不必要的。[[concepts/selective-rag|选择性 RAG]] 让模型自行判断是否需要跨文件信息，避免不必要检索。

## 相关

- [[techniques/sparse-vs-dense-retrieval]] — 两种检索范式的详细对比
- [[techniques/bm25-retrieval]] — PL→PL 场景的最优选择
- [[concepts/retrieval-quality]] — 检索质量受任务模态影响
- [[concepts/selective-rag]] — PL→PL 场景中检索必要性较低，选择性 RAG 尤其适用
