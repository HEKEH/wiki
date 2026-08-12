---
title: Token 经济学（成本与延迟）
date: 2026-08-11
tags: [cost, latency, token, production, optimization]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md, Code-Execution-with-MCP.md, OpenAI-A-Practical-Guide-to-Building-Agents.md]
---

# Token 经济学（成本与延迟）

Agent 的性能、成本、延迟三者由同一个变量支配：**token**。能把这层关系讲清楚，面试里"你们怎么控制成本"就不会答成空话。

## 基本事实（可引用的数字）

- **token 用量单独解释 BrowseComp 上 80% 的性能方差**；加上工具调用次数与模型选择，三者共解释 95%
- **agent ≈ chat 的 4× token；多 agent ≈ 15×**
- 模型升级是"效率倍增器"：**升级到 Sonnet 4 带来的提升大于把 Sonnet 3.7 的 token 预算翻倍**
- 代码执行替代直接 tool call：**150,000 → 2,000 token（-98.7%）**

第一条与第二条合起来是关键判断：**多花 token 确实能换性能，但只有任务价值足够高才值得。**

## 成本来源与对应手段

| 成本来源 | 手段 |
|---|---|
| 工具定义占满 context | 渐进披露、`search_tools`、代码执行（[[concepts/32-progressive-disclosure-and-code-execution]]） |
| 中间结果反复穿过 context | 在执行环境里过滤/聚合后再返回；子 agent 蒸馏后回传（1k–2k token） |
| 历史无限增长 | compaction、清理历史工具结果、结构化笔记（[[concepts/30-compaction-and-note-taking]]） |
| 工具响应过大 | 分页/过滤/截断 + 合理默认（Claude Code 默认 25,000 token 上限） |
| 每个任务都用最强模型 | **先用最强模型建基线，再用小模型替换验证** —— 分级路由（简单检索/意图分类用小模型，退款审批这类难判断用强模型） |
| 重复前缀反复计费 | prompt caching（把稳定的 system prompt/工具定义放前面，让缓存生效） |
| 串行等待 | 并行工具调用、并行 subagent（复杂查询研究时间最多降 90%）、把条件分支写进代码省"首 token 延迟" |
| 无界循环 | `max_turns`、限流与熔断（也对应 OWASP **LLM06 Unbounded Consumption**，2026 年上升 4 位） |

## 努力缩放（effort scaling）

不要让所有查询消耗同等资源。把规则写进 prompt：

- 简单事实：1 个 agent，3–10 次工具调用
- 直接比较：2–4 个 subagent，各 10–15 次
- 复杂研究：10+ subagent，职责明确划分

早期常见失败正是**给简单查询派 50 个 subagent**。反向失败是复杂任务投入不足、过早收工（参见 [[concepts/06-context-anxiety]]）。

## 与注意力预算的关系

token 同时是**钱**和**注意力**（[[concepts/29-context-rot-and-attention-budget]]）。这带来一个很有用的判断：

> 大多数省 token 的手段同时提升准确率，因为它们减少的是低信号 token。

反例也存在：过度压实会丢关键细节，过度截断会让 agent 反复重查 —— 反而更贵。所以要用 [[concepts/41-agent-evaluation]] 的指标（工具调用次数、错误率、总 token、成功率）一起看。

## 面试回答模板

1. 先讲**度量**：每任务 token/成本/延迟、缓存命中率、工具调用分布
2. 再讲**架构级**手段：渐进披露 / 代码执行 / 子 agent 蒸馏 / 混合检索
3. 再讲**参数级**手段：模型分级、缓存、并行、截断与分页、努力缩放规则
4. 最后给**取舍判据**：任务价值 vs 15× 成本；延迟敏感 → 预检索，质量优先 → agentic search

## 来源

- [[sources/07-multi-agent-research-system]]、[[sources/09-code-execution-with-mcp]]、[[sources/05-openai-practical-guide-to-building-agents]]、[[sources/06-effective-context-engineering]]
