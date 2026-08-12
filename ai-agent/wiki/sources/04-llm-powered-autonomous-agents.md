---
title: "LLM Powered Autonomous Agents (Lilian Weng, 2023)"
date: 2026-08-11
tags: [source, survey, planning, memory, tool-use, foundational]
sources: [LLM-Powered-Autonomous-Agents.md]
status: ingested
---

# LLM Powered Autonomous Agents

- 作者：[[entities/08-lilian-weng]]（时任 OpenAI Safety Systems 负责人）
- 发表：2023-06-23，Lil'Log
- 原文：[raw/LLM-Powered-Autonomous-Agents.md](../../raw/LLM-Powered-Autonomous-Agents.md) ｜ 配图 [raw/assets/llm-powered-autonomous-agents/](../../raw/assets/llm-powered-autonomous-agents/)

Agent 领域**被引用最多的中文/英文入门综述**。它在 AutoGPT 热潮期把散落的论文归纳成一张架构图，此后几乎所有 Agent 面试题的知识框架都源自这篇文章。想快速建立"AI Agent 由哪几块组成"的心智模型，从这里开始。

## 核心论断：Agent = LLM 大脑 + 三大组件

文章把 LLM-powered agent 拆成三个组件，这个三分法已成事实标准：

| 组件 | 内涵 | 代表工作 |
|---|---|---|
| **Planning**（规划） | 子目标分解 + 反思改进 | CoT、ToT、LLM+P、ReAct、Reflexion |
| **Memory**（记忆） | 短期 = in-context；长期 = 外部向量库 | MIPS / ANN 检索 |
| **Tool use**（工具） | 调用外部 API 弥补权重内缺失的能力 | MRKL、TALM、Toolformer、HuggingGPT |

对应 [[concepts/14-augmented-llm]]：Anthropic 的"增强 LLM"（检索 + 工具 + 记忆）与这里的三组件是同一件事的两种表述。

## Planning：两条支线

1. **任务分解**——[[concepts/22-planning-and-decomposition]]
   - CoT：`think step by step`，把大任务变成一串小任务
   - ToT：每步生成多个候选思路，构成树，用 BFS/DFS + 评估器搜索
   - 分解的三种来源：LLM 自己提示分解、任务专用指令（"写小说大纲"）、人类输入
   - LLM+P：把规划外包给经典规划器（PDDL），LLM 只做自然语言 ↔ PDDL 的翻译
2. **自我反思**——[[concepts/23-react]]、[[concepts/24-reflection]]
   - ReAct：动作空间 = 任务动作 + 语言空间，`Thought → Action → Observation` 循环；实验证明比只有 `Act` 的基线更好
   - Reflexion：二元奖励 + 启发式函数判定轨迹"低效"或"幻觉"（连续相同动作导致相同观察），失败后写反思进工作记忆（最多 3 条）再重试
   - Chain of Hindsight / Algorithm Distillation：把"带反馈的历史输出序列"喂进 context，让模型顺着改进趋势继续

## Memory：人脑类比 → 工程映射

- 感觉记忆 ≈ 原始输入的 embedding 表示
- 短期/工作记忆 ≈ in-context learning，受 context window 限制
- 长期记忆 ≈ 外部向量库 + 快速检索

长期记忆的工程核心是 **MIPS（最大内积搜索）**，实践中用 ANN 近似：LSH、ANNOY（随机投影树）、HNSW（分层小世界图）、FAISS（向量量化 + 聚类）、ScaNN（各向异性量化）。这是 [[concepts/27-rag]] 与 [[concepts/26-memory]] 的技术底座，也是面试中"向量检索为什么快"的标准答案来源。

## Tool use：从 MRKL 到 HuggingGPT

- **MRKL**：LLM 作路由器，把请求分发给"专家模块"（神经或符号，如计算器、汇率 API）。实验结论很重要——**知道何时用、如何用工具，比工具本身更难**，取决于模型能力
- **TALM / Toolformer**：微调让模型自己学会调 API（以"加入 API 调用是否提升输出质量"来扩充数据集）
- **ChatGPT Plugins / function calling**：工业化落地形态，即今天的 [[concepts/25-tool-use-and-function-calling]]
- **HuggingGPT** 四阶段：任务规划 → 模型选择 → 任务执行 → 响应生成
- **API-Bank** 三级评测：能否调用 API → 能否检索 API → 能否规划多次调用，是 [[concepts/42-agent-benchmarks]] 的早期范式

## 案例研究

- **ChemCrow**（13 个化学专家工具）：LLM 评估认为它与 GPT-4 相当，人类专家评估却认为它大幅胜出 —— **LLM 在需要深度专业知识的领域不适合当裁判**，这是 LLM-as-judge 的经典反例，见 [[concepts/41-agent-evaluation]]
- **Generative Agents**（25 个虚拟角色）：memory stream + 检索（相关性/新近性/重要性三因子）+ 反思 + 规划，见 [[concepts/44-generative-agents]]
- **AutoGPT / GPT-Engineer**：文章附了完整 system prompt，是研究"早期 agent 如何用提示词硬撑格式解析"的一手材料

## 三大挑战（2023 年提出，至今仍是主线）

1. **有限 context length** —— 检索能扩容但表达力不如 full attention。今天的答案是 [[concepts/29-context-rot-and-attention-budget]] + [[concepts/30-compaction-and-note-taking]]
2. **长程规划与纠错困难** —— 遇到意外错误时不擅长调整计划
3. **自然语言接口不可靠** —— 格式错误、偶发抗命，大量 agent 代码都在做输出解析。今天由结构化输出/工具调用 API 部分解决，参见 [[concepts/21-agent-loop]]

## 与本库其他来源的关系

- 这篇是**学术视角**的自底向上综述（论文驱动）；[[sources/03-building-effective-ai-agents]] 是**工程视角**的自顶向下分类（模式驱动）；两者搭配是最佳入门组合
- 文中"finite context"挑战 → [[sources/06-effective-context-engineering]] 给出了 2025 年的系统答案
- 文中"long-term memory = 向量库"这一默认答案，已被 Anthropic 的 agentic search / 文件系统记忆部分替代，见 [[concepts/28-agentic-search]]
