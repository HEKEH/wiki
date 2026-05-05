---
tags: [agent, workflow, architecture, classification]
date: 2026-04-25
sources:
  - sources/Building-Effective-AI-Agents.md
---

# Agentic Systems 分类框架

Anthropic 提出的统一分类：将所有 Agent 变体统称为 **Agentic Systems**，并在架构层面区分 Workflows 和 Agents。

## 定义

- **Workflows**：LLM 和工具沿**预定义代码路径**编排的系统。开发者设计控制流，LLM 在节点内执行。
- **Agents**：LLM **动态控制**自身流程和工具使用的系统。LLM 自主决定下一步做什么。

## 核心区别

| 维度 | Workflows | Agents |
|------|-----------|--------|
| 控制流 | 预定义，代码决定 | 动态，LLM 决定 |
| 可预测性 | 高 | 低 |
| 灵活性 | 受限于预设路径 | 可处理开放性问题 |
| 适用场景 | 任务可拆解为固定子步骤 | 难以预测步骤数的开放任务 |
| 成本/延迟 | 可控 | 较高，可能复合错误 |

## 选择原则

1. **能用单次 LLM 调用解决的，不要用 Agent** — 加检索和示例就够了
2. **需要可预测性时选 Workflow** — 明确定义的任务
3. **需要灵活性时选 Agent** — 模型驱动的决策
4. **只在简单方案不足时增加复杂度** — 复杂度必须有验证过的收益

## 五种 Workflow 模式

从简到繁递进：

1. [Prompt Chaining](prompt-chaining.md) — 串行分解
2. [Routing](routing.md) — 分类路由
3. [Parallelization](parallelization.md) — 并行分片/投票
4. [Orchestrator-Workers](orchestrator-workers.md) — 动态编排
5. [Evaluator-Optimizer](evaluator-optimizer.md) — 迭代优化

这些模式可自由组合，不是固定处方。

### Workflow 模式在 Agent 中的对应

五种模式并非"只属于 Workflow"——它们在 Agent 中同样出现，但形态发生了根本转换：从"开发者预设的代码路径"变成"LLM 在运行时自主选择的行为"。

| Workflow 模式 | 在 Agent 中的表现 | 关键差异 |
|---|---|---|
| [Prompt Chaining](prompt-chaining.md) | Agent 读完 issue → 规划 → 改文件 → 跑测试，天然链式推进 | 链的长度和内容由 LLM 动态决定，不是代码写死的 |
| [Routing](routing.md) | Agent 根据子任务性质选择工具：文件编辑器 vs Shell vs 浏览器 | 分类器就是 LLM 自己，不是独立的分类步骤 |
| [Parallelization](parallelization.md) | Claude Code 可以并行发 Bash 调用；子 agent 并行工作 | 文章中的 Agent 图只画了串行循环，但实际实现支持并行 |
| [Orchestrator-Workers](orchestrator-workers.md) | Agent 就是编排器，工具就是 worker | Workflow 版的 worker 是独立 LLM 实例；Agent 版的 worker 是工具调用 |
| [Evaluator-Optimizer](evaluator-optimizer.md) | Coding Agent 跑测试、看失败、迭代修改 | 评估由工具（测试框架）完成，不是独立评估器 LLM |

核心洞察：**五种 Workflow 模式是"冻结的设计决策"**——开发者在编码时决定何时串行、何时路由、何时并行。**Agent 则"融化"了这些模式**——LLM 在运行时自主做出同样的决策。这也印证了 [The Bitter Lesson](bitter-lesson.md)：预定义的 Workflow 逻辑是"人类知识的特定设计"，随着模型能力增长，可能消融为 Agent 的动态行为。

例外：[Parallelization](parallelization.md) 的 Voting 变体在 Agent 中较少见——Agent 通常不会对自己的输出投票。这是更偏 Workflow 层面的模式。

### "中心决策+分发"谱系

[Routing](routing.md) 和 [Orchestrator-Workers](orchestrator-workers.md) 结构同源——都是"中心决策 → 分发到专门处理"，区别在于**选一个还是分多个**：

- **Routing 是互斥选择**——走 A 就不走 B，分支之间是"或"关系
- **Orchestrator-Workers 是组合分解**——任务拆成互补子任务，都需完成，结果是"与"关系

两者可视为同一个谱系在不同参数下的实例：

```
谱系：中心决策 + 分发

  固定 1-of-N         动态 1-of-N        动态 M-of-N
  ──────────────→ ──────────────→ ──────────────→
  硬编码路由           Routing          Orchestrator-Workers
  (if/else)         (LLM 分类)          (LLM 编排)
```

- 最左边：代码写 `if/else`，完全预定义
- Routing：用 LLM 分类代替 if/else，但分支仍是预定义的完整处理
- Orchestrator-Workers：不仅分类用 LLM，**子任务本身也是 LLM 动态决定的**

Orchestrator-Workers 本质是 Routing 的泛化——把"选一个分支"升级为"按需拆解并组合多个子任务"。

## 与 Wiki 其他概念的关系

- [Harness](harness.md) 是 Workflow 和 Agent 的**运行时实现**：控制循环 + 工具路由 + 状态管理
- [Brain-Hands-Session 解耦模型](brain-hands-session.md) 是 Agentic System 的**物理部署架构**
- [The Bitter Lesson](bitter-lesson.md) 暗示：随着模型能力增长，Workflows 中硬编码的逻辑可能逐步消融为纯 Agent

## 来源

- [Building Effective AI Agents](../sources/Building-Effective-AI-Agents.md) — Anthropic 工程博客
