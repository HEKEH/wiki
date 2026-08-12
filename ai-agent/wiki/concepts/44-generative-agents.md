---
title: Generative Agents 与技能库（模拟与终身学习）
date: 2026-08-11
tags: [generative-agents, voyager, memory, simulation, papers]
sources: [LLM-Powered-Autonomous-Agents.md, Foundational-Agent-Papers-Abstracts.md]
---

# Generative Agents 与技能库

两篇"非工具型 agent"的经典工作。它们的价值不在直接落地，而在提供了**记忆架构**与**技能积累**的原型 —— 这两点后来都进入了工程实践。

## Generative Agents（2304.03442）

25 个由 LLM 驱动的虚拟角色在沙盒（灵感来自 The Sims）中生活与互动。四个机制：

1. **Memory stream（记忆流）**：外部数据库，用自然语言完整记录 agent 的经历；每个元素是一条 **observation**；agent 之间的交流会触发新的自然语言陈述
2. **Retrieval model（检索模型）**：按三因子给记忆打分并取回 ——
   - **Recency（新近性）**：近期事件得分更高
   - **Importance（重要性）**：区分琐事与核心记忆，**直接问 LLM 打分**
   - **Relevance（相关性）**：与当前情境/查询的相关程度
3. **Reflection（反思）**：随时间把记忆**综合成更高层的推断**。做法：用最近 100 条观察提示 LLM 生成 3 个最显著的高层问题，再让 LLM 回答它们。⚠️ 注意这与 [[concepts/24-reflection]] 里的"纠错式自我反思"是**不同含义**
4. **Planning & Reacting**：把反思与环境信息转成行动；环境信息以树结构呈现；规划要在"当下可信"与"长期可信"之间取平衡

结果：涌现出社会行为 —— 信息扩散、关系记忆（两个 agent 延续之前的话题）、社交活动协调（办派对并邀请他人）。

**工程启示**：三因子检索比纯向量相似度更接近"回忆"；把"重要性打分"交给 LLM 是可复用技巧（见 [[concepts/26-memory]]）。

## Voyager（2305.16291）

Minecraft 中首个 LLM 驱动的**终身学习**具身 agent，三个组件：

1. **自动课程（automatic curriculum）**：最大化探索，自己决定下一个该学什么
2. **不断增长的技能库（skill library）**：**存储可执行代码**来保存与检索复杂行为
3. **迭代式 prompting 机制**：结合环境反馈、执行错误、**自我验证**来改进程序

它通过黑盒查询 GPT-4，**不做参数微调**。技能是时序延展的、可解释的、可组合的 —— 能力因此复利式增长。

**工程启示**：这正是 [[concepts/32-progressive-disclosure-and-code-execution]] 里 "agent 把跑通的代码存成 `./skills/*.ts` + `SKILL.md`" 的学术前身。也解释了为什么 Anthropic 说 agent 能"随时间积累出更高层能力的工具箱，自己演化脚手架"。

## 与 AutoGen（2308.08155）

同期的多 agent 编排框架：以"可对话 agent"为基本单元，agent 可组合、可插入人类。它与 [[concepts/35-manager-vs-decentralized]] 讨论的形态相互印证 —— 多 agent 的核心抽象是**消息传递 + 角色分工**。

## 为什么面试还值得提这两篇

- 被问"记忆怎么设计"时，三因子检索是能立刻说出的具体方案
- 被问"agent 能不能自己变强"时，技能库 + 自我验证是有论文支撑的答案，且能连到 Skills 的当代实现
- 两篇都展示了**不依赖微调**也能获得持续能力增长，契合 [[concepts/08-bitter-lesson]] 的讨论

## 来源

- [[sources/04-llm-powered-autonomous-agents]]、[[sources/15-foundational-agent-papers]]、[[sources/09-code-execution-with-mcp]]
