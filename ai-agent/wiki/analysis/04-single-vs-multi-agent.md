---
title: 单 Agent vs 多 Agent：决策框架
date: 2026-08-11
tags: [analysis, multi-agent, architecture, decision]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md, OpenAI-A-Practical-Guide-to-Building-Agents.md, Effective-Context-Engineering-for-AI-Agents.md]
---

# 单 Agent vs 多 Agent：决策框架

这是 agent 架构面试**最常见的开放题**。好回答的结构是：默认立场 → 触发条件 → 代价 → 反面情形 → 具体形态选择。

## 默认立场：先把单 agent 做满

两家厂商罕见地完全一致：

- **OpenAI**："我们的一般建议是先最大化单个 agent 的能力。更多 agent 能带来直观的概念分离，但也引入额外复杂度与开销，通常一个带工具的 agent 就够了。"
- **Anthropic**（[[concepts/13-agentic-systems]]）：找最简单的方案，只在必要时增加复杂度。

## 触发拆分的四个信号

| 信号 | 说明 | 出处 |
|---|---|---|
| **复杂逻辑** | prompt 里出现大量 if-then-else 分支，模板难扩展 | OpenAI |
| **工具过载** | 不是数量问题而是**相似/重叠度**：有系统管好 15+ 个界限清晰的工具，也有栽在 10 个重叠工具上 | OpenAI |
| **context 装不下** | 信息量超单个 context window，需要并行独立探索 | Anthropic |
| **需要并行加速** | 任务天然可切成独立子问题（广度优先型查询） | Anthropic |

拆分前先试：改工具名/描述/参数、拆 prompt 模板、加策略变量。**能靠 ACI 改进解决的，不要靠架构解决。**

## 代价（必须主动说出来）

- **成本**：agent ≈ chat 的 **4×** token；多 agent ≈ **15×** → 需要任务价值支撑
- **协调复杂度爆炸**：早期失败包括给简单查询派 50 个 subagent、无休止搜索不存在的来源、互相用过多更新干扰
- **涌现行为**：改 lead 的 prompt 会不可预测地改变 subagent 行为，评估必须看**交互模式**
- **同步瓶颈**：lead 同步等待 subagent，单个慢 subagent 阻塞全局
- **信息损耗**："传话游戏" —— 需要让 subagent 直接写文件、只回传引用来缓解

## 明确不适合多 agent 的情形

Anthropic 原文点名：

> 需要**所有 agent 共享同一 context**、或 agent 间**依赖很多**的领域，目前不适合多 agent。例如**大多数编码任务的可并行部分远少于研究任务**，而且 LLM 目前不擅长实时协调与委派。

反过来，适合的是：**高价值 + 重并行 + 信息量超单 context + 需对接大量复杂工具**（研究、大范围信息核查、跨源比对）。

## 决策流程图（可在白板上画）

```text
任务能用一次 LLM 调用 + 检索/示例解决？
  ├─ 是 → 别做 agent（优化单次调用）
  └─ 否 → 路径可预先固定？
       ├─ 是 → 用 workflow 模式（chaining / routing / parallel / orchestrator / evaluator）
       └─ 否 → 单 agent + 工具
            ├─ 工具重叠混乱 or prompt 分支爆炸？→ 先修 ACI，再考虑拆
            ├─ 需要综合多方结果？→ Manager / agents-as-tools（orchestrator-worker）
            ├─ 只需转给专家独立处理？→ Handoff（去中心化）
            ├─ context 装不下 or 需并行探索？→ Subagent 架构（干净 context + 蒸馏回传）
            └─ 跨组织/跨框架且对端不透明？→ A2A
```

## 一个容易被忽略的中间选项

**子 agent 不等于多 agent 系统**。[[concepts/30-compaction-and-note-taking]] 把 subagent 列为**上下文管理技术**之一：主 agent 持计划，子 agent 用干净 context 深挖后回传 1k–2k token 摘要。这种用法的目的不是"分工"，而是**隔离 context 污染**，成本远低于完整的多 agent 编排。

面试里这是很好的加分点：**"我会先用子 agent 做上下文隔离，而不是直接上多 agent 架构。"**

## 相关

- [[concepts/34-orchestrator-worker-multi-agent]]：生产级多 agent 的细节与数字
- [[concepts/35-manager-vs-decentralized]]：两种形态的机制差异
- [[concepts/33-a2a]]：跨组织协作的协议层
- [[analysis/03-framework-comparison]]：框架选型
