---
title: Orchestrator-Worker 多 Agent 系统
date: 2026-08-11
tags: [multi-agent, orchestration, subagents, token-economics]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md, Effective-Context-Engineering-for-AI-Agents.md]
---

# Orchestrator-Worker 多 Agent 系统

Anthropic Research 功能的架构，也是**生产级多 agent 最有据可查的形态**。与 [[concepts/18-orchestrator-workers]]（作为 workflow 模式的版本）的区别在于：这里的 lead 是真 agent，worker 数量与任务在运行时由模型决定。

## 架构

```text
用户 query
  → LeadResearcher：分析 → 制定策略 → 把计划写入 Memory（防 context 截断）
  → 并行派生 Subagent（各自独立 context、工具、轨迹）
      → 各自搜索 + interleaved thinking 评估结果 → 回传蒸馏发现
  → Lead 综合 → 判断是否需要更多研究（可再派或调整策略）
  → CitationAgent：为每个论断定位引用
  → 返回带引用的报告
```

## 为什么有效：三个可引用的数字

1. **+90.2%**：Opus 4 lead + Sonnet 4 subagent 相对单 agent Opus 4 的内部 research eval 提升
2. **token 用量单独解释 80% 的性能方差**（BrowseComp）；加上工具调用次数与模型选择共解释 95%
3. 机制解释：**多 agent 之所以有效，主要因为它能花掉足够多的 token**；子 agent 各有独立 context，等于扩容了并行推理容量

另一个机制视角：**搜索的本质是压缩**。子 agent 并行探索不同侧面，把最重要的 token 蒸馏给 lead；同时提供关注点分离（各自工具/prompt/轨迹），降低路径依赖。

## 代价与不适用

- **成本**：agent ≈ chat 的 **4×** token；多 agent ≈ **15×** → 只有任务价值足够高才经济
- **不适合**：需要所有 agent 共享同一 context、或 agent 间依赖很多的领域
- 原文点名：**大多数编码任务的可并行部分远少于研究任务**，且 LLM 目前不擅长实时协调与委派
- **适合**：高价值 + 重并行 + 信息量超单 context + 需对接大量复杂工具

## 让它工作的工程细节

- **委派要具体**：每个 subagent 需要目标、输出格式、工具与来源指引、清晰边界。反例"research the semiconductor shortage" → 3 个 subagent 有 1 个跑去查 2021 年汽车芯片危机，另 2 个重复查 2025 供应链
- **努力缩放规则写进 prompt**：简单事实 1 agent / 3–10 次调用；直接比较 2–4 agent 各 10–15 次；复杂研究 10+ agent 且职责明确划分（早期常见失败是给简单查询派 50 个 subagent）
- **两层并行**：lead 并行起 3–5 个 subagent；每个 subagent 并行用 3+ 工具 → 复杂查询研究时间最多降 90%
- **子 agent 输出落盘**：只回传轻量引用，避免"传话游戏"与 token 复制
- **同步执行是当前瓶颈**：lead 同步等待 subagent，简化协调但阻塞信息流（lead 无法中途操纵、subagent 之间无法协调）；异步能提升并行度，但带来结果协调、状态一致性、错误传播难题
- **涌现行为**：改 lead 的 prompt 会不可预测地改变 subagent 行为 → 必须评估**交互模式**而非单体

## 与其他多 agent 形态的关系

| 形态 | 控制权 | 出处 |
|---|---|---|
| **Orchestrator-worker（本页）** | lead 持有，worker 只回结果 | Anthropic Research |
| **Manager / agents-as-tools** | manager 持有，子 agent 是工具 | OpenAI（等价形态） |
| **Decentralized / handoffs** | **转移**给被移交的 agent | OpenAI |
| **A2A** | 跨组织的对等协作，对端不透明 | [[concepts/33-a2a]] |

见 [[concepts/35-manager-vs-decentralized]] 与 [[analysis/04-single-vs-multi-agent]]。

## 来源

- [[sources/07-multi-agent-research-system]]、[[sources/06-effective-context-engineering]]
