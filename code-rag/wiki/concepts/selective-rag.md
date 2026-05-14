---
title: Selective RAG
date: 2026-05-12
tags: [selective-rag, adaptive-retrieval, retrieval-policy, inference-efficiency]
sources: [Repoformer-Selective-Retrieval-for-Repository-Level-Code-Completion.md]
---

# Selective RAG

选择性检索增强生成——系统主动判断是否需要检索，而非总是执行检索。

## 核心洞察

**并非所有查询都需要检索**。[[sources/src-repoformer-selective-retrieval|Wu et al. (2024)]] 发现，在仓库级代码补全中：
- 仅 ~20% 检索实例提升性能
- ~60% 无影响
- ~20% 反而损害性能

这与 [[concepts/retrieval-quality|检索质量]] 的传统视角形成互补——不仅存在"检索得好不好"的问题，更存在"该不该检索"的决策问题。

## 决策策略分类

| 策略 | 机制 | 延迟开销 | 效果 |
|------|------|---------|------|
| **Trial Retrieval** | 先检索，按相似度分数决定是否使用 | 无延迟节省（仍需检索） | 可节省 40-50% 检索预算 |
| **Trial Generation** | 先生成，按不确定性决定是否检索 | 需额外推理 | 对长序列生成失效 |
| **Self-assessment** | 模型自评估是否需要检索 | 极低（单次 forward pass） | 最优——同时提升精度和延迟 |

## Self-assessment 的实现

[[systems/repoformer|Repoformer]] 的方法：模型在 `<eof>` token 后预测特殊 token `<cc>` 的概率，超过阈值则触发检索。这本质上是让模型结合两个信号：
1. **自知性** (self-knowledge)：模型是否已经知道答案（Kadavath et al., 2022）
2. **任务依赖性**：问题是否依赖跨文件信息

## 效率收益

Selective RAG 的延迟节省来自两个维度：
1. **跳过检索**：当模型弃权检索时，完全避免检索器的延迟
2. **并行决策**：在 online serving 设置中，决策进程与生成进程并行执行

在 Dense retrieval + 大仓库场景下，20% 的检索预算可转化为 **70% 的推理加速**。

## 与 Always-RAG 的权衡

| 维度 | Always-RAG | Selective RAG |
|------|-----------|---------------|
| 精度 | 基线 | ≥ 基线（避免有害检索） |
| 延迟 | 基线 | 显著降低 |
| 鲁棒性 | 受噪声检索影响 | 更鲁棒（训练后学会过滤噪声） |
| 实现复杂度 | 简单 | 需训练自评估能力 |

## 相关概念

- [[concepts/retrieval-quality]] — Selective RAG 是提升检索质量的新维度
- [[concepts/pl-pl-vs-nl-pl-retrieval]] — PL→PL 场景中检索的必要性更低（代码补全常可从当前文件推断）
- [[systems/repoformer]] — Selective RAG 的代表性系统
