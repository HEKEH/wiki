---
tags: [workflow, agent, pattern, iteration, feedback]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Evaluator-Optimizer

一个 LLM 生成响应、另一个 LLM 评估反馈的循环迭代 Workflow 模式。

## 机制

```
Input → [Generator LLM] → [Evaluator LLM] → 满足标准？
              ↑                                    │
              └──────── Feedback ───────────── No ─┘
                                                   │
                                              Yes → Output
```

- 生成器和评估器用不同的 prompt，关注不同方面
- 评估器提供具体、可操作的反馈，而非仅评分
- 循环直至评估器判定满足标准

## 何时使用

- 有**明确的评估标准**
- 迭代改进能带来**可衡量的价值**
- 两个信号：① 人类反馈能改善 LLM 输出；② LLM 能提供此类反馈

## 典型场景

- 文学翻译：翻译 LLM 可能遗漏细微含义，评估 LLM 提供有用批评
- 复杂搜索任务：需多轮搜索分析，评估器判断是否需要继续搜索

## 与其他模式的关系

- 可与 [Prompt Chaining](prompt-chaining.md) 组合：串行步骤中嵌入评估循环
- 可与 [Orchestrator-Workers](orchestrator-workers.md) 组合：编排器委派后评估结果
- 与 [Feature List Pattern](feature-list-pattern.md) 精神一致：都有明确的完成标准和迭代验证

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
