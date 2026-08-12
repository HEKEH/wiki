---
title: Manager 模式 vs 去中心化模式
date: 2026-08-11
tags: [multi-agent, orchestration, handoff, patterns]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, OpenAI-Agents-SDK-Documentation.md]
---

# Manager 模式 vs 去中心化模式

OpenAI 归纳的两大**广泛适用**的多 agent 类别。核心区别只有一句话：**控制权是否转移。**

## 对照

| | **Manager（agents as tools）** | **Decentralized（handoffs）** |
|---|---|---|
| 机制 | 中心 manager 通过 **tool call** 调用专家 agent | agent 之间**单向移交**执行权，连同最新会话状态 |
| 控制权 | 始终在 manager | 转移给被移交者 |
| 谁面对用户 | 只有 manager | 接管者直接与用户交互 |
| 图论视角 | 边 = tool call | 边 = handoff |
| 适合 | 需要一个 agent 统一控制流程并综合结果 | 对话分流（triage）、专家完全接管某类任务 |
| 代码形态 | `spanish_agent.as_tool(tool_name=..., tool_description=...)` 放进 manager 的 `tools` | `Agent(..., handoffs=[support, sales, orders])` |

两者都可建模为图（agent 是节点）。**无论用哪种，原则相同：保持组件灵活、可组合，由清晰而结构良好的 prompt 驱动。**

## 选择判据

先问：**这个任务需要有人"综合"吗？**

- 需要综合（翻译成三种语言后合并、多源研究后写报告）→ **Manager**
- 不需要综合，只需要"转给对的人"（客服分流到技术支持/销售/订单）→ **Decentralized**

再问：**上下文要不要跟着走？** handoff 会转移会话状态，接管者能看到完整对话；agents-as-tools 则由 manager 决定给子 agent 看什么（信息隔离更强，但可能丢上下文）。

## Manager 模式的两个变体

- **纯工具化**：子 agent 完全不知道自己在被编排，只收到结构化输入 → 便于测试与替换
- **Orchestrator-worker**：lead 是真 agent，动态决定派多少 worker、各做什么（[[concepts/34-orchestrator-worker-multi-agent]]）

## 何时**别**拆多 agent

OpenAI 的默认建议：**先把单 agent 的能力做满**。触发拆分的两个信号：

1. **复杂逻辑**：prompt 里出现大量 if-then-else 分支，模板难以扩展 → 按逻辑段拆
2. **工具过载**：关键不是数量而是**相似/重叠度**（有系统管好 15+ 个清晰工具，也有栽在 10 个重叠工具上）→ 先改名/改描述/改参数，无效再拆

见 [[analysis/04-single-vs-multi-agent]]。

## 配套能力（生产会用到）

- **approval gates**：给 agent-as-tool 加审批门（[[concepts/38-human-in-the-loop]]）
- **自定义输出抽取**：从子 agent 运行结果里只取需要的部分，控制回传 token
- **结构化输入**：给 agent-as-tool 定义输入 schema，减少歧义
- **流式嵌套运行**：把子 agent 的过程事件透传给 UI

## 与其他概念的关系

- [[concepts/36-handoff]]：去中心化模式的机制细节
- [[concepts/16-routing]]：单 LLM 分类路由，是 decentralized 模式的 workflow 版本（不转移控制权，只选分支）
- [[concepts/18-orchestrator-workers]]：manager 模式的 workflow 版本

## 来源

- [[sources/05-openai-practical-guide-to-building-agents]]、[[sources/11-openai-agents-sdk-docs]]
