---
title: "Foundational Agent Papers (arXiv 摘要汇编)"
date: 2026-08-11
tags: [source, papers, arxiv, reasoning, benchmarks]
sources: [Foundational-Agent-Papers-Abstracts.md]
status: ingested
---

# Foundational Agent Papers：28 篇基础论文摘要汇编

- 来源：arXiv abs 页面抓取的元数据与摘要原文，按主题分组
- 原文：[raw/Foundational-Agent-Papers-Abstracts.md](../../raw/Foundational-Agent-Papers-Abstracts.md)

面试中被问"你读过哪些 agent 论文"时，需要的不是全文，而是**每篇一句话的贡献 + 它解决了什么问题 + 它的局限**。这一页给出速查表，展开见各概念页。

## 推理与规划 → [[concepts/22-planning-and-decomposition]]

| 论文 | 一句话贡献 |
|---|---|
| **CoT** (2201.11903) | 提供少量"思维链"范例即可显著提升算术/常识/符号推理，能力随规模涌现 |
| **Self-Consistency** (2203.11171) | 采样多条推理路径取多数答案，替代贪心解码 |
| **Tree of Thoughts** (2305.10601) | 把推理组织成树，每步生成多个 thought 并用评估器 + BFS/DFS 搜索，可回溯 |
| **Plan-and-Solve** (2305.04091) | 零样本下先出计划再按计划执行，缓解 zero-shot CoT 的漏步/计算错 |
| **PAL** (2211.10435) | 推理步骤写成**程序**，把计算交给解释器执行 —— "会算的事别让模型算" |

## 工具使用与行动 → [[concepts/23-react]]、[[concepts/25-tool-use-and-function-calling]]

| 论文 | 一句话贡献 |
|---|---|
| **ReAct** (2210.03629) | 交错生成 reasoning trace 与 action，推理指导行动、行动反馈推理，奠定 agent 主循环范式 |
| **Toolformer** (2302.04761) | 自监督地让模型学会何时调哪个 API、传什么参数、如何用返回值 |
| **MRKL** (2205.00445) | 神经-符号模块化架构，LLM 作路由分发给专家模块；点明"知道何时/如何用工具"才是难点 |
| **ReWOO** (2305.18323) | 把推理与观察**解耦**：先一次性规划全部工具调用（无需等观察），再执行、再求解，大幅省 token |
| **LLM Compiler** (2312.04511) | 把函数调用编译成 DAG 并行执行，降低延迟与成本 |

## 自我改进与反思 → [[concepts/24-reflection]]

| 论文 | 一句话贡献 |
|---|---|
| **Reflexion** (2303.11366) | 用**语言**而非权重更新做强化：把失败轨迹的自我反思写入 episodic memory，下轮作为上下文 |
| **Self-Refine** (2303.17651) | 同一个 LLM 生成 → 自评反馈 → 自我修订的迭代，无需额外训练 |

## 记忆与检索 → [[concepts/26-memory]]、[[concepts/27-rag]]

| 论文 | 一句话贡献 |
|---|---|
| **RAG** (2005.11401) | 参数化记忆 + 非参数化检索（DPR + 生成器）联合，开创检索增强范式 |
| **MemGPT** (2310.08560) | 借操作系统的虚拟内存分页思想管理 context：主上下文 ↔ 外部存储自主换页 |
| **RAG Survey** (2312.10997) | Naive / Advanced / Modular RAG 三阶段分类，检索-生成-增强三维度综述 |
| **GraphRAG** (2404.16130) | 用 LLM 建实体知识图 + 社区摘要，回答需要全局理解的 query-focused summarization |
| **Memory Survey** (2404.13501) | 系统梳理 agent 记忆的来源、形式、操作与评测 |

## 多智能体与环境交互 → [[concepts/44-generative-agents]]、[[concepts/34-orchestrator-worker-multi-agent]]

| 论文 | 一句话贡献 |
|---|---|
| **Generative Agents** (2304.03442) | 25 个虚拟角色的沙盒；memory stream + 检索（相关性/新近性/重要性）+ 反思 + 规划，涌现社会行为 |
| **Voyager** (2305.16291) | Minecraft 中的终身学习 agent：自动课程 + **技能库（可执行代码）** + 迭代式 prompting 自我验证 |
| **AutoGen** (2308.08155) | 可对话 agent 的多 agent 编排框架，agent 可组合、可插人类 |

## 综述

- **A Survey on LLM based Autonomous Agents** (2308.11432)：构建（profile/memory/planning/action）、应用、评测三视角
- **The Rise and Potential of LLM Based Agents** (2309.07864)：brain-perception-action 框架 + 单 agent / 多 agent / 人机协作三类场景

## 评测基准 → [[concepts/42-agent-benchmarks]]

| 基准 | 测什么 |
|---|---|
| **SWE-bench** (2310.06770) | 真实 GitHub issue → 修复补丁，测跨文件仓库级软件工程能力 |
| **τ-bench** (2406.12045) | 工具-agent-**用户**三方交互，动态对话 + 领域规则遵循 + `pass^k` 可靠性指标 |
| **GAIA** (2311.12983) | 需要推理 + 多模态 + 浏览 + 工具的真实助手任务（人类 92% vs GPT-4+插件 15%） |
| **AgentBench** (2308.03688) | 8 个环境下把 LLM 当 agent 评（多轮开放式生成） |
| **WebArena** (2307.13854) | 可复现的真实网站环境，端到端评网页任务 |

## 安全

- **Not what you've signed up for**（2302.12173）：**间接提示注入**的开山之作 —— 攻击者把 prompt 放进会被检索的数据里，远程影响 LLM 集成应用。今天 OWASP LLM01 的理论基础，见 [[concepts/39-prompt-injection]]

## 阅读优先级建议

面试冲刺只精读 5 篇：**ReAct → Reflexion → RAG → MemGPT → SWE-bench**，其余按上表记住"一句话贡献"即可。详细路线见 [[analysis/01-interview-roadmap]]。
