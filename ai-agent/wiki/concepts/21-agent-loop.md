---
title: Agent Loop（Agent 主循环）
date: 2026-08-11
tags: [agent, loop, architecture, fundamentals]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, OpenAI-Agents-SDK-Documentation.md, Effective-Context-Engineering-for-AI-Agents.md]
---

# Agent Loop（Agent 主循环）

Agent 的最小定义与实现骨架。Anthropic 现在采用的定义最简洁：

> **Agent = LLM autonomously using tools in a loop.**（LLM 在循环中自主使用工具）

## 三大组件（OpenAI 版本）

| 组件 | 内容 |
|---|---|
| **Model** | 驱动推理与决策的 LLM |
| **Tools** | 可调用的外部函数/API |
| **Instructions** | 定义行为的显式指南与护栏 |

任何编排方案都需要 **run（一次运行）** 的概念：一个循环，让 agent 运转直到触发退出条件。

## 循环的标准形态

```text
loop:
  1. 把 (instructions, 历史, 工具定义) 发给 LLM
  2. LLM 输出：
     - 纯文本且无工具调用  → 视为最终输出，结束
     - 工具调用            → 执行工具，把结果追加进历史，continue
     - handoff（若支持）   → 切换当前 agent 与输入，continue
  3. 若超过 max_turns → 抛错终止
```

OpenAI Agents SDK 的 `Runner.run()` 就是这个循环的直接实现；判定"最终输出"的规则是**产生了期望类型的文本输出，且没有工具调用**。LangGraph 的等价物是 `llm_call` 节点 + 工具节点 + 条件边构成的环。

## 退出条件（面试常问）

- 产生最终输出（无工具调用）
- 调用了指定的 final-output 工具 / 产出指定的结构化输出类型
- 出错
- 达到最大轮次上限（`max_turns`）
- 外部中断：人工介入、超时、取消（`cancel(mode="after_turn")`）、预算耗尽

**为什么必须有上限**：agent 会陷入循环（连续相同动作 → Reflexion 把这定义为一种"幻觉"）；无界循环还直接对应 OWASP **LLM06 Unbounded Consumption** 的成本风险。

## 让循环稳定的四个旋钮

1. **结构化输出**：用 schema 约束输出类型（`output_type` / `with_structured_output`），把"解析模型输出"这件早期 agent 最大的工程负担交给 API。Lilian Weng 2023 年列的三大挑战之一"自然语言接口不可靠"主要靠它缓解
2. **工具结果的 token 预算**：Claude Code 默认把单次工具响应截断在 25,000 token，见 [[concepts/45-token-economics]]
3. **每轮的 context 策展**：循环每转一圈都在往 context 里加东西，因此 [[concepts/09-context-engineering]] 是循环稳定性的核心，长程要配 [[concepts/30-compaction-and-note-taking]]
4. **护栏与人工介入**：并发运行的输入/输出校验（[[concepts/37-guardrails]]）+ 高风险动作前的审批（[[concepts/38-human-in-the-loop]]）

## 与 workflow 的界线

- **Workflow**：路径由代码预先编排（[[concepts/13-agentic-systems]] 的五种模式）
- **Agent**：路径由模型在循环中动态决定
- 判据不是"用了几个 LLM"，而是**谁在控制流程**。OpenAI 的说法：不用 LLM 控制工作流执行的（简单 chatbot、单轮调用、分类器）都不是 agent

## 与其他概念的关系

- [[concepts/02-harness]] 是这个循环的工程实现体（含工具路由、会话、恢复）
- [[concepts/23-react]] 是这个循环最早的 prompt 形态（Thought/Action/Observation）
- [[concepts/25-tool-use-and-function-calling]] 是循环第 2 步的机制细节
- [[concepts/43-durable-execution]] 讨论循环跑几小时后如何不丢状态

## 来源

- [[sources/05-openai-practical-guide-to-building-agents]]、[[sources/11-openai-agents-sdk-docs]]、[[sources/06-effective-context-engineering]]
