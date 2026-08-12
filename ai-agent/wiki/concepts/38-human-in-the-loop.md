---
title: Human-in-the-Loop（人工介入）
date: 2026-08-11
tags: [hil, safety, production, approval]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, OWASP-GenAI-LLM-Top-10-2026.md, LangGraph-Workflows-and-Agents.md]
---

# Human-in-the-Loop（人工介入）

不是"降级方案"，而是**让 agent 在不牺牲用户体验的前提下改进真实表现的关键保障**，在部署早期尤其重要 —— 它帮助识别失败、发现边界情况、建立稳健的评估循环。

## 两个标准触发器（OpenAI）

1. **超过失败阈值**：为重试次数或动作次数设上限；超限（例如多次仍无法理解客户意图）就升级到人
2. **高风险动作**：敏感、不可逆、影响大的操作 —— 取消订单、批大额退款、发起支付；在对 agent 可靠性的信心增长前保持人工监督

## 分级执行策略（OWASP LLM03 的"complete mediation"）

比"全都要审批"更实用的做法：**audit → warn → block → escalate** 分级。

> 低后果或易回滚的动作自动放行，高后果或不可逆的动作路由到人工复核。例：客服机器人可自动办理**店铺积分**形式的退款（可回收），但**对外打款**这类不可逆动作转人工。

配套判据（与 [[concepts/37-guardrails]] 的 tool safeguards 一致）：按**只读 vs 写入、可逆性、所需权限、资金影响**给每个工具打风险分。

## 实现形态

- **审批门（approval gate）**：工具执行前暂停等待批准（Agents SDK 对 agent-as-tool 提供 approval gates）
- **中断与恢复**：LangGraph 的 `interrupt` + checkpointer —— 在任意点检查/修改 agent 状态再继续，这需要状态可持久化（[[concepts/43-durable-execution]]）
- **优雅移交**：agent 无法完成任务时把控制权交回 —— 客服场景转人工坐席，编码 agent 把控制权还给用户（OpenAI 对 agent 的定义里就包含"失败时能中止并交回控制权"）
- **异步审批**：长时任务里把待批准动作入队，人批完再继续（Temporal/Restate/Dapr/DBOS 这类持久执行框架的典型用途）

## 两个反直觉的注意点

1. **审批疲劳会削弱判断**：OWASP 明确提示大量审批会降低复核质量 → 所以要**分级**，而不是一律弹窗
2. **给人看的必须是"真实将执行的动作"而非摘要**：不可见 Unicode 走私可能让**显示的动作与实际执行的动作不一致**（OWASP LLM01 缓解 #7）

## 与评估的关系

人工评估捕捉自动化遗漏的问题：Anthropic 的人类测试者发现早期 agent 一贯偏好 SEO 内容农场而非权威但排名低的学术 PDF —— 这类**系统性偏差**几乎不可能被自动 eval 发现。见 [[concepts/41-agent-evaluation]]。

## 与其他概念的关系

- [[concepts/40-excessive-agency]]：HIL 是治"excessive autonomy"的核心手段
- [[concepts/37-guardrails]]：护栏判定"要不要拦"，HIL 决定"谁来放行"
- [[concepts/03-sandbox]]：另一种不依赖人的边界控制

## 来源

- [[sources/05-openai-practical-guide-to-building-agents]]、[[sources/14-owasp-genai-llm-top-10-2026]]、[[sources/12-langgraph-workflows-and-agents]]、[[sources/07-multi-agent-research-system]]
