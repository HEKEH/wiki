---
title: Handoff（执行权移交）
date: 2026-08-11
tags: [handoff, multi-agent, routing, agents-sdk]
sources: [OpenAI-Agents-SDK-Documentation.md, OpenAI-A-Practical-Guide-to-Building-Agents.md]
---

# Handoff（执行权移交）

去中心化多 agent 的核心机制。**实现上它就是一种工具**：

> In the Agents SDK, a handoff is a type of tool, or function. If an agent calls a handoff function, we immediately start execution on that new agent while also transferring the latest conversation state.

## 三个性质

1. **单向**：A 移交给 B 后，A 不再参与（除非 B 也被赋予"移交回 A"的 handoff）
2. **转移会话状态**：B 拿到最新的对话上下文，能无缝接着聊
3. **在 agent loop 内即时生效**：runner 检测到 handoff 就切换 current agent 与输入，然后重跑循环（见 [[concepts/21-agent-loop]]）

## 典型形态：Triage

```python
triage_agent = Agent(
    name="Triage Agent",
    instructions="你是第一接触点，评估客户问题并及时转给正确的专家 agent。",
    handoffs=[technical_support_agent, sales_assistant_agent, order_management_agent],
)
```

用户问"最近那笔订单什么时候到" → triage 识别为订单问题 → handoff 给 order_management_agent → 由它接管并回答。

## handoff vs agent-as-tool vs routing

| | 控制权 | 上下文 | 谁给最终答复 |
|---|---|---|---|
| **handoff** | 转移 | 跟着走 | 接管的 agent |
| **agent-as-tool** | 不转移 | 由 manager 决定给多少 | manager |
| **routing**（[[concepts/16-routing]]） | 不涉及 agent，只是代码分支 | 由代码决定 | 被路由到的处理逻辑 |

选择顺序建议：能用 routing 解决就别用 agent；需要专家独立对话就用 handoff；需要综合多方结果就用 agent-as-tool。

## 设计注意

- **handoff 的描述就是路由的 prompt**：写清"什么情况该转给我"，与工具描述同等重要（[[concepts/20-aci]]）
- **避免 handoff 乱跳**：给出明确边界与"不属于我时该转给谁"，否则会出现 A→B→A 的抖动
- **要保留原始用户授权范围**：移交后仍以该用户的最小权限执行下游动作，别退化成"服务身份"（OWASP LLM03，见 [[concepts/40-excessive-agency]]）
- **可观测性**：把一次逻辑对话中的多次 agent 切换聚合到同一 trace（`group_id`/workflow name），否则排查极难（[[concepts/43-durable-execution]]）

## 来源

- [[sources/11-openai-agents-sdk-docs]]、[[sources/05-openai-practical-guide-to-building-agents]]
