---
title: Repoformer
date: 2026-05-12
tags: [selective-rag, code-completion, starcoder, self-assessment, repository-level]
sources: [Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# Repoformer

基于 StarCoderBase 微调的仓库级代码补全模型，核心创新是 [[concepts/selective-rag|选择性 RAG]]——模型自行判断是否需要跨文件检索。

## 架构

Repoformer 将选择性检索决策嵌入 fill-in-the-middle (FIM) 流程：

```
<fim_prefix> [Left Context] <fim_suffix> [Right Context] <eof> → <cc> or φ
                                                          ↓ (if <cc>)
                                              [Cross-file Context]
                                                          ↓
                                             <fim_middle> [Completion]
```

单次 left-to-right pass 完成"评估→（可选）检索→生成"全流程。

## 模型变体

| 变体 | 基座 | 关键特点 |
|------|------|---------|
| Repoformer-1B | StarCoderBase-1B | 可作为即插即用策略模型加速更大 LM |
| Repoformer-3B | StarCoderBase-3B | 性能超 5× 大的 StarCoder-16B (部分 ES) |
| Repoformer-7B | StarCoderBase-7B | — |
| Repoformer-16B | StarCoder-16B | 所有任务 SOTA |

另有 Python/Java/C#/TypeScript 多语言版本。

## 训练

- **数据**：The Stack 18K Python 仓库，240K chunk + 120K function 实例
- **标注**：对比有无 CC 的 Edit Similarity 差值
- **损失**：$\lambda \mathcal{L}_{eval} + \mathcal{L}_{gen}$（$\lambda=1.0$）
- **硬件**：8× A100 (40G)，1B 模型 ~8h，16B 模型 ~50h

## 检索配置

- **检索器**：Jaccard 相似度（也验证了 dense retrieval 兼容性）
- **Chunk size**：20 行（line/API/chunk 补全）/ 50 行（function 补全）
- **Overlap**：stride = chunk_size / 2
- **Top-k**：10 + Fragment Alignment

## 性能亮点

- Selective 策略比 Always-RAG 同尺寸 StarCoderBase 平均高 3+ ES
- Greedy selection：仅 ~20% 实例检索，60-70% 推理加速，-1 ES
- Threshold selection：~60-80% 实例检索，16-28% 加速，**精度正提升**
- 作为策略模型加速其他 LM：~25% 加速 + 精度提升

## 基准

- **RepoEval**：line/API/function 补全（32 Python 仓库）
- **CrossCodeEval**：line 补全（4 语言）
- **CrossCodeLongEval**（新）：chunk/function 补全（1500 Python 仓库）

## 相关

- [[concepts/selective-rag]] — Repoformer 的核心概念
- [[sources/src-repoformer-selective-retrieval]] — 论文详细摘要
- [[techniques/sparse-vs-dense-retrieval]] — Repoformer 使用 Jaccard 稀疏检索
- [[techniques/code-chunking]] — 索引分块策略
