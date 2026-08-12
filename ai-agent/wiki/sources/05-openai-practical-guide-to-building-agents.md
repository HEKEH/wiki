---
title: "A Practical Guide to Building Agents (OpenAI)"
date: 2026-08-11
tags: [source, openai, orchestration, guardrails, engineering]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, OpenAI-A-Practical-Guide-to-Building-Agents.pdf]
status: ingested
---

# A Practical Guide to Building Agents

- 作者：[[entities/07-openai]]（34 页官方白皮书，面向产品与工程团队）
- 原文：[raw/OpenAI-A-Practical-Guide-to-Building-Agents.pdf](../../raw/OpenAI-A-Practical-Guide-to-Building-Agents.pdf)（文本版 [.md](../../raw/OpenAI-A-Practical-Guide-to-Building-Agents.md)）

与 [[sources/03-building-effective-ai-agents]] 并列的两大厂商指南之一。Anthropic 那篇偏"模式分类学"，这篇偏"落地清单"：**什么时候该做 agent、三大组件怎么配、编排怎么选、护栏怎么搭**。面试中被问"你们怎么设计 agent 系统"，这篇的骨架最好背。

## Agent 的定义与判据

> Agents are systems that independently accomplish tasks on your behalf.

不是 agent 的：简单 chatbot、单轮 LLM 调用、情感分类器 —— **凡是没有用 LLM 控制工作流执行的，都不算**。判定 agent 的两条核心特征：

1. 用 LLM 管理工作流执行与决策：能识别任务何时完成、能主动纠错、失败时能中止并把控制权交回用户
2. 能访问工具与外部系统，并**在明确护栏内**根据当前状态动态选择工具

## 什么时候该建 agent

优先挑那些"传统自动化一直搞不定"的工作流：

| 场景 | 说明 | 例子 |
|---|---|---|
| 复杂决策 | 需要细致判断、例外处理、上下文敏感 | 客服退款审批 |
| 规则难以维护 | 规则集庞杂，改一处代价高、易出错 | 供应商安全审查 |
| 重度依赖非结构化数据 | 需要理解自然语言、从文档抽取语义 | 房屋保险理赔 |

反面判据同样重要：**如果确定性方案够用，就别做 agent**（与 [[concepts/13-agentic-systems]] 的"简单优先"一致）。文中用支付欺诈举例：规则引擎像清单，LLM agent 像资深调查员。

## 三大基础组件

一个 agent 最小构成 = **Model + Tools + Instructions**，见 [[concepts/21-agent-loop]]。

- **选模型**：不是每个任务都要最强模型。方法论三步 ——（1）先建 eval 拿到基线；（2）用最强模型达到准确率目标；（3）再用小模型替换以优化成本延迟。**先建立性能上限，再压成本**，避免过早限制能力
- **定工具**：三类工具 —— **Data**（取上下文：查库、读 PDF、搜网）、**Action**（写操作：发邮件、更新 CRM、转人工）、**Orchestration**（agent 本身作为工具，即 manager 模式）。工具要有标准化定义、文档齐全、可复用，见 [[concepts/25-tool-use-and-function-calling]] 与 [[concepts/20-aci]]。遗留系统没有 API 时，可用 computer-use 模型直接操作 UI
- **写指令**：最佳实践 —— 复用既有 SOP/政策文档生成 routine；提示 agent 拆解任务；每一步对应一个明确动作或输出；显式覆盖边界情况（信息缺失、意外提问的分支）。可以用推理模型把帮助中心文档自动转成编号指令

## 编排：单 agent 优先

**Single-agent systems**：一个模型 + 合适工具 + 指令，在循环中执行。"run"的概念是核心 —— 循环直到退出条件：工具调用完成、产出特定结构化输出、报错、或达到最大轮次上限（Agents SDK 的 `Runner.run()`：产生最终输出 或 模型不再请求工具调用）。

控制复杂度的技巧：**用一个带策略变量的 prompt 模板**代替维护几十个 prompt（新场景改变量而非重写流程）。

**什么时候拆多 agent**（默认先把单 agent 能力做满）：
- **复杂逻辑**：prompt 里出现大量 if-then-else 分支，模板难以扩展
- **工具过载**：关键不是工具数量而是**相似/重叠度** —— 有的系统能管好 15+ 个界限清晰的工具，有的却栽在 10 个重叠工具上。先尝试改名/改描述/改参数，无效再拆

## 两种多 agent 形态

| 模式 | 机制 | 适用 |
|---|---|---|
| **Manager（agents as tools）** | 中心 manager 通过 tool call 调度专家 agent，自己综合结果 | 只希望一个 agent 控制流程并面对用户 |
| **Decentralized（handoffs）** | 平级 agent 之间**单向移交**执行权，连同最新会话状态 | 对话分流（triage），专家 agent 完全接管 |

多 agent 系统可建模为图：manager 模式的边是 tool call，去中心化模式的边是 handoff。见 [[concepts/35-manager-vs-decentralized]] 与 [[concepts/36-handoff]]。

文中还有一段针对**声明式 vs 非声明式框架**的立场：声明式框架要求预先画出所有分支/循环（可视化清晰，但工作流一动态化就笨重，还要学 DSL）；Agents SDK 选择 code-first，用熟悉的编程结构直接表达逻辑，无需预定义整张图。这正是 [[analysis/03-framework-comparison]] 的核心分歧点。

## Guardrails：分层防御

> 单一护栏不足以提供充分保护，多个专用护栏叠加才能造出更有韧性的 agent。

七类护栏（见 [[concepts/37-guardrails]]）：relevance classifier（跑题）、safety classifier（越狱/注入）、PII filter（输出侧个人信息）、moderation（有害内容）、**tool safeguards**（按只读/写入、可逆性、权限、资金影响给每个工具打低/中/高风险分，高风险触发暂停或转人工）、rules-based protections（黑名单、长度限制、正则）、output validation（品牌一致性）。

建设顺序的启发式：先做数据隐私与内容安全 → 根据真实边界情况和失败逐步加 → 同时优化安全与体验。Agents SDK 把护栏当一等概念，默认**乐观执行**：主 agent 照常产出，护栏并发跑，违规就抛异常（tripwire）。

## 规划人工介入

两个触发器（见 [[concepts/38-human-in-the-loop]]）：
1. **超过失败阈值**：重试/动作次数超限（例如多次仍无法理解客户意图）就升级到人
2. **高风险动作**：敏感、不可逆、影响大的操作 —— 取消订单、批大额退款、发起支付

结论一句话：**Start small, validate with real users, grow capabilities over time.**
