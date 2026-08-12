---
title: 规划与任务分解（Planning & Task Decomposition）
date: 2026-08-11
tags: [planning, reasoning, cot, tot, papers]
sources: [LLM-Powered-Autonomous-Agents.md, Foundational-Agent-Papers-Abstracts.md]
---

# 规划与任务分解

[[sources/04-llm-powered-autonomous-agents]] 三大组件之一。核心问题：**复杂任务需要多步，agent 得先知道有哪些步、以什么顺序做。**

## 谱系：从提示技巧到显式搜索

| 方法 | 机制 | 代价 / 局限 |
|---|---|---|
| **CoT**（2201.11903） | "think step by step"，用测试时计算把大任务拆成小步 | 单条路径，一步错则全错 |
| **Self-Consistency**（2203.11171） | 采样多条推理路径，取答案多数投票 | N 倍成本；只适合有唯一答案的题 |
| **Tree of Thoughts**（2305.10601） | 每步生成多个候选 thought 构成树，BFS/DFS + 评估器（prompt 打分或投票）搜索，可回溯 | 成本高，需要设计状态评估器 |
| **Plan-and-Solve**（2305.04091） | 零样本下先产出计划再执行，缓解 zero-shot CoT 的漏步 | 计划一旦错，执行照错 |
| **PAL**（2211.10435） | 推理写成程序，计算交给解释器 | 只适合可程序化的子问题 |
| **LLM+P**（2304.11477） | 翻译成 PDDL 交给经典规划器，再翻译回自然语言 | 需要领域 PDDL 与可用规划器（机器人常见，其他领域少） |
| **ReWOO**（2305.18323） | **推理与观察解耦**：先一次性规划所有工具调用，再执行、再求解 | 无法根据中间观察调整计划 |
| **LLM Compiler**（2312.04511） | 把函数调用编译成 DAG 并行执行 | 依赖任务可并行 |

## 分解的三种来源（Lilian Weng）

1. LLM 自提示分解：`"Steps for XYZ.\n1."`、`"What are the subgoals for achieving XYZ?"`
2. 任务专用指令：`"Write a story outline."`
3. 人类输入

## 工程实践：把"计划"当作产物

现代 agent harness 的做法不是让模型在 context 里默想计划，而是**把计划外化成可持久、可核对的对象**：

- **写进文件**：progress file / `NOTES.md` / to-do 列表（[[concepts/30-compaction-and-note-taking]]）
- **写成结构化列表**：[[concepts/12-feature-list-pattern]] 把需求拆成细粒度可验证项，一次做一个、端到端测过才标完成
- **在多 agent 中先规划再委派**：lead agent 先规划**并把计划存入 memory**（防 context 截断丢失），再派 subagent（[[concepts/34-orchestrator-worker-multi-agent]]）
- **让努力程度匹配复杂度**：把"简单事实 1 个 agent / 3–10 次调用；复杂研究 10+ subagent"这类缩放规则写进 prompt

## 面试要点

- **CoT / ToT 属于"测试时计算"策略**，本质是用更多 token 换准确率；今天推理模型（extended thinking）把这部分内建化了 —— 这是 [[concepts/08-bitter-lesson]] 的活例证：曾经的 prompt 技巧被模型能力吸收
- **规划失败的两种典型形态**：过度规划（简单任务派 50 个 subagent）与规划僵化（遇到意外错误不调整）。前者用缩放规则约束，后者靠 [[concepts/24-reflection]] 与循环内重规划
- 被问"你怎么让 agent 处理长任务"时，答案层次应是：**分解 → 外化计划 → 增量执行 → 每步验证 → 跨 session 交接**（[[concepts/11-initializer-coding-agent]]）

## 与其他概念的关系

- [[concepts/23-react]] 把规划与行动交错进同一循环
- [[concepts/15-prompt-chaining]] / [[concepts/18-orchestrator-workers]] 是把分解**固化进代码**的 workflow 形态
- [[concepts/19-evaluator-optimizer]] 为计划的产出加上评估回路

## 来源

- [[sources/04-llm-powered-autonomous-agents]]、[[sources/15-foundational-agent-papers]]
