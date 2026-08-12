---
title: ReAct（Reasoning + Acting）
date: 2026-08-11
tags: [react, reasoning, tool-use, papers, foundational]
sources: [Foundational-Agent-Papers-Abstracts.md, LLM-Powered-Autonomous-Agents.md]
---

# ReAct（Reasoning + Acting）

Yao et al., ICLR 2023（[arXiv 2210.03629](https://arxiv.org/abs/2210.03629)）。**当代 agent 主循环的原型**，也是面试出现频率最高的单篇论文。

## 核心思想

把动作空间扩展为「任务专用离散动作 + 语言空间」：
- **语言空间** → 让模型生成推理轨迹（thought）
- **任务动作** → 让模型与环境交互（如查 Wikipedia API）

两者**交错生成**：推理指导下一步行动，行动的观察反过来修正推理。

```text
Thought: 我需要先查 X 的定义
Action: search[X]
Observation: X 是……
Thought: 那么还需要 Y 的数据
Action: search[Y]
...
Answer: ...
```

## 关键实验结论

- 在知识密集任务（HotpotQA、FEVER）与决策任务（ALFWorld、WebShop）上，**ReAct 都优于去掉 `Thought` 的 Act-only 基线**
- 推理轨迹还带来**可解释性与可诊断性**：人能看出它为什么走错

## 为什么它重要

1. 它是"LLM 在循环中用工具"的最早完整表述 —— 今天的 [[concepts/21-agent-loop]] 就是 ReAct 的工程化版本
2. 它统一了两条此前分离的研究线：CoT（只推理，不与外界交互，易幻觉）和工具调用（只行动，无推理规划）
3. 它把**观察结果重新进入 context** 这一机制固定下来 —— 这也是后来 [[concepts/39-prompt-injection]] 中"工具输出即不可信输入"的结构根源

## 今天还需要手写 ReAct 吗？

**通常不需要**，这是重要的加分回答：

- 现代模型的 **function calling / tool use API** 原生支持"边想边调工具"，不必再用 `Thought:/Action:/Observation:` 的文本协议去解析（早期 agent 大量代码都在做格式解析）
- **extended thinking / interleaved thinking** 把 thought 变成模型的一等能力：Anthropic 的做法是 lead agent 用 thinking 规划、subagent 在工具结果后用 interleaved thinking 评估质量与差距
- 但 ReAct 的**结构**仍然在：思考 → 行动 → 观察 → 再思考。手写它的场景只剩：用不支持工具调用的模型、需要极强可控性、或做教学/复现

## 变体与后继

- **Reflexion**（[[concepts/24-reflection]]）：在 ReAct 的动作空间上加一层失败后的语言反思与重试
- **ReWOO**：反其道而行 —— 把推理与观察**解耦**以省 token（先规划全部调用再执行）
- **LLM Compiler**：把调用编译成 DAG 并行，解决 ReAct 串行慢的问题
- **Plan-and-Execute 类**：先整体规划再逐步执行，减少每步都重新推理的开销

## 与其他概念的关系

- [[concepts/22-planning-and-decomposition]] 提供 thought 阶段的策略
- [[concepts/25-tool-use-and-function-calling]] 是 action 阶段的现代实现
- [[concepts/17-parallelization]] 与 ReWOO/LLM Compiler 一起构成"ReAct 太慢"的解法族

## 来源

- [[sources/15-foundational-agent-papers]]、[[sources/04-llm-powered-autonomous-agents]]
