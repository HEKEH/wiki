---
tags: [workflow, agent, pattern, parallel]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Parallelization

并行化 Workflow 模式，有两种变体：分片（Sectioning）和投票（Voting）。

## 机制

### Sectioning（分片）

```
Input → [Subtask A] →\
       [Subtask B] →→ Aggregator → Output
       [Subtask C] →/
```

将任务拆为独立子任务并行运行，结果聚合。

### Voting（投票）

```
Input → [LLM instance 1] →\
       [LLM instance 2] →→ Vote/Aggregate → Output
       [LLM instance 3] →/
```

同一任务多次执行，获取多样输出后综合判断。

## 何时使用

- 分片：子任务可并行以提速，或任务有多个关注点需分别处理
- 投票：需要多视角/多次尝试以获得更高置信度

## 典型场景

### Sectioning

- 护栏：一个模型处理用户查询，另一个筛查不当内容——比同一调用兼顾两者效果更好
- 自动化评估：每个 LLM 调用评估模型性能的不同维度

### Voting

- 代码漏洞审查：多个不同 prompt 各自审查，发现问题时标记
- 内容合规判断：多个 prompt 评估不同方面，设置不同投票阈值平衡误报/漏报

## 与其他模式的关系

- 与 [Routing](routing.md) 互补：Routing 选一条路走，Parallelization 同时走多条路
- [Orchestrator-Workers](orchestrator-workers.md) 是 Parallelization 的动态版本：子任务不预定义，由编排器决定

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
