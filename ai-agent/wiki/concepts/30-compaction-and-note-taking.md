---
title: Compaction、结构化笔记与子 Agent（长程三术）
date: 2026-08-11
tags: [context-engineering, long-horizon, compaction, memory, sub-agents]
sources: [Effective-Context-Engineering-for-AI-Agents.md, How-We-Built-Our-Multi-Agent-Research-System.md]
---

# Compaction、结构化笔记与子 Agent

任务 token 量超过 context window 时的三种标准技术。Anthropic 把它们并列给出，并明确了各自的适用边界 —— **这张对照表是长程 agent 面试题的最佳答案骨架**。

## 三术对照

| 技术 | 机制 | 最适合 |
|---|---|---|
| **Compaction（压实）** | 把接近上限的会话总结后，用摘要重启新 context | 需要大量来回的对话流 |
| **Structured note-taking（结构化笔记）** | 定期把笔记写到 context 之外，稍后读回 | 有清晰里程碑的迭代开发 |
| **Sub-agent 架构** | 子 agent 用干净 context 深挖，只回传蒸馏结论 | 并行探索有回报的研究/分析 |

## Compaction

- Claude Code 的实现：把消息历史交给模型总结压缩 —— **保留**架构决策、未解决的 bug、实现细节；**丢弃**冗余工具输出与消息；继续时带上压缩后的 context + **最近访问的 5 个文件**
- 调优方法（很实用）：**先最大化 recall**（确保压实 prompt 不漏任何相关信息），**再提升 precision**（去掉多余内容）。在复杂 agent trace 上调
- **最轻量且最安全的形式：清理工具调用结果** —— "工具在历史深处早就调过了，agent 为什么还要看到原始结果？"（Claude 开发者平台已作为 context management 特性提供）
- 风险：过度压实会丢掉"当时不显眼、后来才重要"的细节

## 结构化笔记（agentic memory）

- 形态：to-do 列表、`NOTES.md`、progress file
- 极低开销地获得持久记忆：跨几十次工具调用维持关键上下文与依赖
- **Claude 玩 Pokémon** 是经典例证：跨数千步维持精确计数（"过去 1,234 步我在 Route 1 练级，皮卡丘已升 8 级，目标 10 级"）、自发绘制探索过的地图、记录哪些招式对哪些对手有效；**context 重置后读回自己的笔记**继续多小时的训练或地牢探索
- 平台支持：Claude 的 **memory tool**（基于文件系统，public beta）
- 与 [[concepts/11-initializer-coding-agent]] 的 progress file + feature list + git history 三重机制是同一思想

## 子 Agent 架构

- 主 agent 持高层计划，子 agent 用**干净 context** 做深度技术工作或信息查找
- 子 agent 可能烧几万 token 探索，**只回传 1,000–2,000 token 的蒸馏摘要**
- 收益：清晰的关注点分离 —— 详细搜索上下文封闭在子 agent 内，lead 专注综合分析
- 附加技巧（避免"传话游戏"）：让子 agent **直接把成果写入文件系统**，只把轻量引用回传给协调者 —— 减少信息损失与 token 复制开销，特别适合代码、报告、可视化这类结构化产出
- 代价与边界见 [[concepts/34-orchestrator-worker-multi-agent]]（15× token）

## 组合使用（生产实际）

真实的长程系统往往三者并用：

1. 平时用**笔记**记录进度与决策
2. 接近窗口上限时**压实**，或**派生带干净 context 的新子 agent** 并做精心交接
3. 需要广度时**并行子 agent**，结果落盘再汇总
4. 跨 session 靠**外部产物**（progress file、feature list、git history、记忆文件）衔接 → [[concepts/10-long-running-agent]]

## 与其他概念的关系

- [[concepts/29-context-rot-and-attention-budget]]：三术存在的根本原因
- [[concepts/01-session]]：笔记与压实产物的持久化载体
- [[concepts/12-feature-list-pattern]]：结构化笔记的一种强约束形式

## 来源

- [[sources/06-effective-context-engineering]]、[[sources/07-multi-agent-research-system]]
