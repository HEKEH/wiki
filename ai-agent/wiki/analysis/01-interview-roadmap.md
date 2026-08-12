---
title: AI Agent 面试学习路线图
date: 2026-08-11
tags: [analysis, roadmap, interview, study-plan]
sources: [Building-Effective-AI-Agents.md, LLM-Powered-Autonomous-Agents.md, OpenAI-A-Practical-Guide-to-Building-Agents.md]
---

# AI Agent 面试学习路线图

面向**尽快达到应对面试水平**的系统学习路径。全部内容都在本知识库内，`raw/` 有一手原文，`wiki/` 有组织好的要点。

## 面试考什么（先建立地图）

一场 AI Agent 岗位面试的问题基本落在六个板块：

| 板块 | 权重 | 核心页面 |
|---|---|---|
| **A. 基础概念与分类** | ★★★★★ | [[concepts/13-agentic-systems]]、[[concepts/21-agent-loop]]、[[concepts/14-augmented-llm]] |
| **B. 编排模式** | ★★★★★ | [[concepts/15-prompt-chaining]]–[[concepts/19-evaluator-optimizer]]、[[concepts/35-manager-vs-decentralized]]、[[concepts/34-orchestrator-worker-multi-agent]] |
| **C. 上下文与记忆** | ★★★★★ | [[concepts/09-context-engineering]]、[[concepts/29-context-rot-and-attention-budget]]、[[concepts/30-compaction-and-note-taking]]、[[concepts/26-memory]]、[[concepts/27-rag]]、[[concepts/28-agentic-search]] |
| **D. 工具与协议** | ★★★★☆ | [[concepts/25-tool-use-and-function-calling]]、[[concepts/20-aci]]、[[concepts/31-mcp]]、[[concepts/32-progressive-disclosure-and-code-execution]]、[[concepts/33-a2a]] |
| **E. 评估与工程化** | ★★★★☆ | [[concepts/41-agent-evaluation]]、[[concepts/42-agent-benchmarks]]、[[concepts/43-durable-execution]]、[[concepts/45-token-economics]] |
| **F. 安全** | ★★★☆☆（但极易挂） | [[concepts/39-prompt-injection]]、[[concepts/40-excessive-agency]]、[[concepts/37-guardrails]]、[[concepts/38-human-in-the-loop]] |
| **G. 论文素养** | ★★☆☆☆ | [[sources/15-foundational-agent-papers]]、[[concepts/22-planning-and-decomposition]]、[[concepts/23-react]]、[[concepts/24-reflection]] |

**经验判断**：A/B/C 决定你能不能过，E/F 决定你是"用过框架"还是"做过系统"，G 只在研究岗或加分环节起作用。

## 14 天计划（每天 2–3 小时）

### 第 1 周：建立骨架 + 一手材料

| 天 | 主题 | 读什么 | 产出（自测） |
|---|---|---|---|
| D1 | 分类框架 | [[sources/03-building-effective-ai-agents]] 全文 + [[concepts/13-agentic-systems]] | 白板画出 workflow vs agent 的分界，说出"何时不该用 agent" |
| D2 | 五种模式 | [[concepts/15-prompt-chaining]]–[[concepts/19-evaluator-optimizer]] + [[sources/12-langgraph-workflows-and-agents]] | 每种模式说一个真实业务例子 + 一段伪代码 |
| D3 | Agent 循环与三组件 | [[sources/05-openai-practical-guide-to-building-agents]] + [[concepts/21-agent-loop]] | 手写 15 行的 agent 主循环（含退出条件） |
| D4 | 综述与论文地图 | [[sources/04-llm-powered-autonomous-agents]] + [[concepts/22-planning-and-decomposition]]、[[concepts/23-react]]、[[concepts/24-reflection]] | 说出 Planning/Memory/Tool 三组件与各自代表工作 |
| D5 | 工具与 ACI | [[sources/08-writing-effective-tools-for-agents]] + [[concepts/25-tool-use-and-function-calling]]、[[concepts/20-aci]] | 把一个 CRUD API 重设计成 3 个 agent 友好的工具 |
| D6 | 上下文工程 | [[sources/06-effective-context-engineering]] + [[concepts/29-context-rot-and-attention-budget]]、[[concepts/30-compaction-and-note-taking]] | 说清 compaction / 笔记 / 子 agent 的适用边界 |
| D7 | RAG 与检索 | [[concepts/27-rag]] + [[concepts/28-agentic-search]] | 画出 RAG 管线并标出 5 个优化点；说出静态 RAG 的三个局限 |

### 第 2 周：深水区 + 面试演练

| 天 | 主题 | 读什么 | 产出（自测） |
|---|---|---|---|
| D8 | 多 agent | [[sources/07-multi-agent-research-system]] + [[concepts/34-orchestrator-worker-multi-agent]]、[[analysis/04-single-vs-multi-agent]] | 背下 90.2% / 80% / 4×·15× 三个数字及其含义 |
| D9 | 协议 | [[sources/10-mcp-architecture-overview]]、[[sources/09-code-execution-with-mcp]]、[[concepts/31-mcp]]、[[concepts/33-a2a]] | 讲清 MCP 三原语、两层、与 function calling / A2A 的区别 |
| D10 | 评估 | [[concepts/41-agent-evaluation]]、[[concepts/42-agent-benchmarks]] | 设计一个 20 条 eval 集 + LLM-as-judge rubric |
| D11 | 安全 | [[sources/16-lethal-trifecta]]、[[concepts/39-prompt-injection]]、[[concepts/40-excessive-agency]] | 复述 lethal trifecta 与 Rule of Two；给一个邮件助手做威胁建模 |
| D12 | 工程化 | [[concepts/43-durable-execution]]、[[concepts/45-token-economics]]、[[concepts/37-guardrails]]、[[concepts/38-human-in-the-loop]] | 讲出"上生产五层清单"与成本优化手段表 |
| D13 | 框架对比 + 长程 | [[analysis/03-framework-comparison]]、[[sources/02-effective-harnesses-for-long-running-agents]]、[[concepts/10-long-running-agent]] | 说出选框架的三条判据；讲清跨 session 交接机制 |
| D14 | 全量刷题 | [[analysis/02-interview-qa-bank]] 全过一遍 + [[analysis/05-glossary]] | 每题限时 90 秒口述，卡住的回去补对应页 |

## 7 天冲刺版（时间紧就走这条）

只做 D1、D3、D5、D6、D8、D11 + 全量刷 [[analysis/02-interview-qa-bank]]。
必读原文压缩到三篇：**Building Effective AI Agents** → **A Practical Guide to Building Agents** → **Effective Context Engineering**。
必背数字：多 agent **+90.2%**、token 解释 **80%** 方差、agent **4×** / 多 agent **15×** token、代码执行 **-98.7%**（150k→2k）、工具响应上限 **25,000** token、起步 eval **约 20 条**、GAIA **人类 92% vs GPT-4+插件 15%**。完整表见 [[analysis/05-glossary]]。

## 三条提分建议

1. **每个概念准备一个"取舍"而不是一个"定义"**。面试官问 workflow 还是 agent、单 agent 还是多 agent、静态 RAG 还是 agentic search —— 想要的是判据与代价，不是名词解释。本库每页末尾的"代价/边界"段就是为此准备的
2. **准备一个自己的项目故事**，能覆盖：架构选择理由 → 遇到的失败模式 → 用什么指标发现 → 怎么改 → 结果。失败模式清单可从 [[analysis/02-interview-qa-bank]] 的"排障"板块取材
3. **数字与出处让回答变得可信**。"多 agent 大约 15 倍 token，所以只在高价值任务上用"比"多 agent 比较贵"强得多

## 手写代码题准备

高频三道：
1. **实现一个最小 agent 循环**（工具调用 + 退出条件 + max_turns）→ [[concepts/21-agent-loop]]
2. **实现 evaluator-optimizer 或 orchestrator-worker**（可用 LangGraph 风格伪代码）→ [[sources/12-langgraph-workflows-and-agents]]
3. **写一个工具定义**并说明为什么这样设计参数与返回（会追问 token 效率与错误信息）→ [[sources/08-writing-effective-tools-for-agents]]

## 相关

- 全部资料清单与扩展阅读：[[analysis/06-source-map]]
- 术语速查：[[analysis/05-glossary]]
