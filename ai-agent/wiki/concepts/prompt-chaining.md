---
tags: [workflow, agent, pattern]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Prompt Chaining

最简单的 Workflow 模式：将任务分解为串行步骤，每个 LLM 调用处理上一步的输出。

## 机制

```
Input → LLM₁ → [Gate?] → LLM₂ → [Gate?] → ... → Output
```

- 每步 LLM 调用专注于更简单的子任务
- 可在步骤间插入**程序化检查**（Gate），确保流程未偏离轨道
- Gate 不需要 LLM，可以是用规则代码判断

## 何时使用

- 任务可**干净地**分解为固定子步骤
- 愿意用延迟换精度——每步 LLM 调用更简单，因此更准确

## 典型场景

- 生成营销文案 → 翻译为另一种语言
- 写文档大纲 → 检查大纲是否符合标准 → 基于大纲写文档

## 在 Workflow 模式中的位置

最基础的编排模式，复杂度最低。当子步骤需要不同处理时，演进为 [Routing](routing.md)；当步骤可并行时，演进为 [Parallelization](parallelization.md)。

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
