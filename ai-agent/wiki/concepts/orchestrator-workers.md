---
tags: [workflow, agent, pattern, orchestration]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Orchestrator-Workers

中心 LLM 动态拆解任务、委派 worker LLM、综合结果的 Workflow 模式。

## 机制

```
Input → [Orchestrator LLM]
          ├→ Worker 1 (specific subtask) →┐
          ├→ Worker 2 (specific subtask) →→ Synthesize → Output
          └→ Worker N (specific subtask) →┘
```

- 编排器根据具体输入**动态决定**需要哪些子任务
- Worker 数量和任务内容由编排器实时确定，不预定义
- 编排器综合所有 worker 结果

## 与 Parallelization 的关键区别

Parallelization 的子任务是**预定义**的；Orchestrator-Workers 的子任务**由编排器动态决定**。

| 维度 | Parallelization | Orchestrator-Workers |
|------|-----------------|---------------------|
| 子任务定义 | 提前确定 | 运行时决定 |
| 灵活性 | 低 | 高 |
| 适用场景 | 固定维度的并行处理 | 无法预测子任务数量和性质 |

## 何时使用

- 复杂任务中**无法预测**所需子任务
- 子任务的数量和性质**取决于具体输入**

## 典型场景

- 编码产品：对多个文件做复杂修改，每次修改的文件数和性质取决于任务
- 搜索任务：从多个来源收集分析信息，无法预知哪些来源相关

## 与 Routing 的谱系关系

Orchestrator-Workers 和 [Routing](routing.md) 结构同源——都是"中心决策 → 分发到专门处理"，区别在于**选一个还是分多个**：

| 维度 | Routing | Orchestrator-Workers |
|------|---------|---------------------|
| 选择 | 1-of-N，互斥 | M-of-N，互补 |
| 分支关系 | 每个分支是完整的端到端处理 | 每个 worker 只做一部分，需综合 |
| 结果 | 直接输出分支结果 | 需要 synthesis 步骤 |
| 子任务定义 | 预定义 | 动态决定 |

两者是"中心决策+分发"谱系上的相邻节点——Orchestrator-Workers 本质是 Routing 的泛化，把"选一个分支"升级为"按需拆解并组合多个子任务"。详见 [Agentic Systems 分类框架](agentic-systems.md) 中的谱系图。

## 与已有概念的关系

- [Harness](harness.md) 的编排逻辑可以视为一种 Orchestrator-Workers 模式
- [Initializer/Coding Agent 模式](initializer-coding-agent.md) 是 Orchestrator-Workers 在长时运行 Agent 中的具体应用
- 在 Agent 中的表现：Agent 本身就是编排器，工具调用就是 worker——Workflow 版的 worker 是独立 LLM 实例，Agent 版的 worker 是工具调用

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
