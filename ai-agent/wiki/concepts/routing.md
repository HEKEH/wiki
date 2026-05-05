---
tags: [workflow, agent, pattern, classification]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Routing

分类输入并路由到专门处理流程的 Workflow 模式，实现关注点分离。

## 机制

```
Input → [Classifier] → Branch A (specialized prompt/tools)
                     → Branch B (specialized prompt/tools)
                     → Branch C (specialized prompt/tools)
```

- 分类器可以是 LLM，也可以是传统分类模型/算法
- 每个分支有专门的 prompt、工具和下游流程
- 关键价值：为一种输入优化不会损害其他输入的性能

## 何时使用

- 复杂任务中有**明显类别**适合分开处理
- 分类可以**准确完成**

## 典型场景

- 客服查询分类：一般问题/退款/技术支持 → 不同流程
- 简单问题路由到小模型（Haiku），难题路由到大模型（Sonnet）——成本优化

## 与 Orchestrator-Workers 的谱系关系

Routing 和 [Orchestrator-Workers](orchestrator-workers.md) 结构同源——都是"中心决策 → 分发到专门处理"，区别在于**选一个还是分多个**：

- **Routing 是互斥选择**（1-of-N）——走 A 就不走 B，分支之间是"或"关系，直接输出分支结果
- **Orchestrator-Workers 是组合分解**（M-of-N）——任务拆成互补子任务，都需完成，需综合（synthesis）步骤

两者可视为"中心决策+分发"谱系上的相邻节点：

```
固定 1-of-N         动态 1-of-N        动态 M-of-N
──────────────→ ──────────────→ ──────────────→
硬编码路由           Routing          Orchestrator-Workers
(if/else)         (LLM 分类)          (LLM 编排)
```

- 硬编码路由：代码写 `if/else`，完全预定义
- Routing：用 LLM 分类代替 if/else，但分支仍是**预定义的完整处理**
- Orchestrator-Workers：不仅分类用 LLM，**子任务本身也是 LLM 动态决定的**

Orchestrator-Workers 本质是 Routing 的泛化——把"选一个分支"升级为"按需拆解并组合多个子任务"。实践中复杂度差异很大：Routing 通常几行代码就够，Orchestrator-Workers 需要更复杂的设计。

## 与其他模式的关系

- 是 [Prompt Chaining](prompt-chaining.md) 的横向扩展：Chaining 是串行，Routing 是分类后分支
- 与 [Parallelization](parallelization.md) 的区别：Routing 选一个分支执行，Parallelization 同时执行多个
- 在 Agent 中的表现：Agent 根据子任务性质选择工具（文件编辑器 vs Shell vs 浏览器），分类器就是 LLM 自身

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
