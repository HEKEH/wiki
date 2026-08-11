---
title: Building Effective AI Agents
tags: [anthropic, agent, workflow, engineering]
date: 2026-04-25
sources: [Building-Effective-AI-Agents.md]
status: ingested
---

# Building Effective AI Agents

Anthropic 工程博客，作者 [[entities/04-erik-schluntz|Erik S.]] 和 [[entities/05-barry-zhang]]。原文链接：<https://www.anthropic.com/engineering/building-effective-agents>

## 核心论点

最成功的 Agent 实现不依赖复杂框架，而是使用**简单、可组合的模式**。从增强型 LLM 出发，只在简单方案不足时才增加复杂度。

## 关键内容

### Agentic Systems 分类

将所有 Agent 变体统称为 [[concepts/13-agentic-systems]]，区分两种架构：

- **Workflows**：LLM 和工具沿预定义代码路径编排
- **Agents**：LLM 动态控制自身流程和工具使用

### 何时使用 Agent

- 优先寻找最简方案，很多场景优化单次 LLM 调用（加检索和示例）就够了
- Workflows 适合有明确定义的任务，需要可预测性和一致性
- Agents 适合需要灵活性和模型驱动决策的场景

### 五种 Workflow 模式

1. [[concepts/15-prompt-chaining]] — 任务拆为串行步骤，每步处理上一步输出，可在中间加检查门控
2. [[concepts/16-routing]] — 输入分类后路由到专门处理流程，实现关注点分离
3. [[concepts/17-parallelization]] — 分片（sectioning）并行处理子任务，或投票（voting）多视角验证
4. [[concepts/18-orchestrator-workers]] — 中心 LLM 动态拆解任务、委派 worker、综合结果
5. [[concepts/19-evaluator-optimizer]] — 一个 LLM 生成，另一个评估反馈，循环迭代

### [[concepts/14-augmented-llm]]

一切 Agent 系统的基础构件：LLM + 检索 + 工具 + 记忆。模型主动生成搜索查询、选择工具、决定保留什么信息。推荐通过 [[concepts/14-augmented-llm|Model Context Protocol]] 集成第三方工具。

### [[concepts/20-aci]]

工具定义应获得与 prompt 工程同等的重视。关键建议：

- 给模型足够 token 在写代码前"思考"
- 格式接近模型在互联网上自然见到的文本
- 避免格式开销（如 diff chunk header 行数计算、JSON 转义）
- 投入与 HCI 同等精力设计 ACI
- [[concepts/20-aci|Poka-yoke]] 工具——防呆设计，如强制绝对路径

### 实践应用（附录）

- **客户支持**：聊天 + 工具集成的天然场景，可按解决率收费
- **Coding Agent**：代码可通过自动化测试验证，Agent 用测试结果迭代；SWE-bench 验证了可行性

### 三大设计原则

1. **保持简单** — Agent 设计中维持简洁
2. **优先透明** — 显式展示 Agent 的规划步骤
3. **精心设计 ACI** — 充分的工具文档和测试

## 与 Wiki 其他内容的关联

- [[concepts/08-bitter-lesson]] 的直接体现：通用简单模式胜过复杂特定设计
- [[concepts/02-harness]] 的上游定义：Workflow 模式就是 Harness 的编排逻辑
- [[concepts/11-initializer-coding-agent]] 与本文 Coding Agent 附录互为补充
- [[concepts/09-context-engineering]] 是 Augmented LLM 中"记忆"增强的具体实践
