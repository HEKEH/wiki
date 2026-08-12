---
title: Context Rot 与注意力预算
date: 2026-08-11
tags: [context-engineering, context-rot, attention, transformer]
sources: [Effective-Context-Engineering-for-AI-Agents.md]
---

# Context Rot 与注意力预算

回答"为什么不能把所有资料都塞进长上下文"的标准依据。

## 现象：Context Rot

> 随着 context window 中 token 数增加，模型准确召回其中信息的能力下降。

- 来自 needle-in-a-haystack 类基准的研究（Chroma 的 context rot 报告）
- **所有模型都有这个性质**，只是衰减陡缓不同
- 结论：**context 是有限资源，且边际收益递减**

## 机制：三个原因

1. **n² 对关系**：Transformer 让每个 token 都能注意到其他所有 token，n 个 token 有 n² 对关系；context 变长时"捕捉这些成对关系的能力被摊薄"，形成 context 大小与注意力聚焦之间的天然张力
2. **训练分布**：训练数据里短序列远多于长序列，模型对**跨全上下文依赖**的经验更少、专用参数更少
3. **位置编码插值**：像 position interpolation 这类让模型处理更长序列的技术，会带来 token 位置理解上的退化

注意准确表述：这些因素造成的是**性能梯度而非硬悬崖** —— 模型在长上下文下仍然很能干，只是相对短上下文，信息检索精度与长程推理会下降。

## 推论：注意力预算（attention budget）

类比人类有限的工作记忆容量：LLM 有一份"注意力预算"，**每个新 token 都在消耗它**。于是 context engineering 的目标可以精确表述为：

> 找到能最大化目标达成概率的、**最小的**高信号 token 集合。

这条判据可以直接用于评审设计：加进 context 的每一样东西（工具定义、示例、历史、检索结果）都要问"它值这份预算吗"。

## 实践含义

- **工具集要剪**：膨胀的工具集不仅占 token，还制造模糊决策点
- **示例要精选**：少量多样、典型的范例 > 边界情况清单
- **工具响应要瘦**：分页/过滤/截断，返回语义化字段而非全量对象
- **历史要治**：清理历史深处的原始工具结果是最安全的压实手段
- **长任务要换架构**：compaction / 笔记 / 子 agent（[[concepts/30-compaction-and-note-taking]]）
- **别指望窗口变大解决问题**：原文明确说，可预见的未来里各种大小的窗口都会受**context 污染与信息相关性**问题影响

## 与其他概念的关系

- [[concepts/09-context-engineering]]：本页是其"为什么重要"的论证
- [[concepts/06-context-anxiety]]：context 将满时模型提前收工的行为，是另一种由窗口限制导致的副作用
- [[concepts/28-agentic-search]]：注意力预算的存在正是"少而准"取代"多而全"的原因
- [[concepts/45-token-economics]]：token 既是成本也是注意力预算，两者常同向优化

## 来源

- [[sources/06-effective-context-engineering]]
