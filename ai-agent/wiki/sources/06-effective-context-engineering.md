---
title: "Effective Context Engineering for AI Agents (Anthropic)"
date: 2026-08-11
tags: [source, anthropic, context-engineering, long-horizon]
sources: [Effective-Context-Engineering-for-AI-Agents.md]
status: ingested
---

# Effective Context Engineering for AI Agents

- 作者：[[entities/01-anthropic]] Applied AI 团队（Prithvi Rajasekaran、Ethan Dixon、Carly Ryan、Jeremy Hadfield）
- 原文：[raw/Effective-Context-Engineering-for-AI-Agents.md](../../raw/Effective-Context-Engineering-for-AI-Agents.md)

把 [[concepts/09-context-engineering]] 从口号变成方法论的文章。一句话总纲：**找到能最大化目标达成概率的、最小的高信号 token 集合**。

## Context engineering vs prompt engineering

- prompt engineering：怎么写好指令（尤其 system prompt），面向一次性分类/生成任务
- context engineering：在推理过程中**策展与维护**整个 token 状态 —— system 指令、工具、MCP、外部数据、消息历史全都算
- 关键区别：写 prompt 是离散动作，**策展是每一轮推理都要重做的循环动作**

## 为什么重要：注意力预算

- **context rot**：token 数增加，模型准确召回信息的能力下降（Chroma 的研究）。所有模型都有这个性质，只是衰减陡缓不同
- 机制：Transformer 中 n 个 token 有 n² 对关系；训练数据里长序列本就稀少，模型对"跨全上下文依赖"的专用参数更少；位置编码插值也带来退化
- 结论：**context 是有限资源，边际收益递减**，模型有"注意力预算"，每个新 token 都在消耗它 → [[concepts/29-context-rot-and-attention-budget]]

## 有效 context 的解剖

- **System prompt 要在"正确的海拔"（right altitude）**：一端是硬编码脆弱 if-else 逻辑，另一端是空泛的高层指导（假装存在共享上下文）。最优是"具体到能引导行为，又灵活到留给模型启发式空间"。用 `<background_information>`、`<instructions>`、`## Tool guidance` 等分节组织；追求**最小但完整**（minimal ≠ short）
- **工具**：token 高效、鼓励高效行为、自包含、抗错、用途极其清晰。最常见失败是**工具集膨胀**——"如果人类工程师都说不清该用哪个工具，agent 更不可能做对"
- **示例（few-shot）**：不要堆边界情况清单，而要策展一组**多样、典型**的范例。"对 LLM 来说，例子就是那张值一千字的图"

## 运行时取 context：agentic search

文中给了 Anthropic 现在采用的 agent 定义：**LLM autonomously using tools in a loop**（引 Simon Willison）。

- 传统做法：推理前用 embedding 检索把资料塞进去
- 新做法 **just-in-time**：只维护轻量标识符（文件路径、查询、链接），运行时用工具动态加载。Claude Code 就是这样对大型数据库做分析：写定向查询、存结果、用 `head`/`tail` 分析，从不把完整数据载入 context
- **元数据本身是信号**：`tests/test_utils.py` 与 `src/core_logic/test_utils.py` 含义不同；文件大小暗示复杂度、命名暗示用途、时间戳代表相关性 → **progressive disclosure**，逐层组装理解
- 代价：运行时探索比预计算慢，且需要良好的工具与启发式，否则 agent 会绕远路
- **混合策略**：Claude Code 把 `CLAUDE.md` 直接放进 context，同时用 glob/grep 即时检索，绕开索引过期与语法树复杂度问题。内容不那么动态的领域（法律、金融）更适合混合
- 详见 [[concepts/28-agentic-search]]

## 长程任务的三种技术

当任务 token 量超过 context window（几十分钟到数小时的连续工作，如大规模迁移、综合研究）：

1. **Compaction（压实）** —— 把接近上限的会话总结后，用摘要重启新 context。Claude Code 的做法：保留架构决策、未解决的 bug、实现细节，丢弃冗余工具输出，再带上最近访问的 5 个文件。调优方法：**先最大化 recall（不漏信息），再提升 precision（去冗余）**。最安全的轻量形式是**清理工具调用结果**（历史深处的原始结果不必再看）
2. **Structured note-taking（结构化笔记 / agentic memory）** —— 定期把笔记写到 context 之外再拉回。如 to-do 列表、`NOTES.md`。Claude 玩 Pokémon 的例子：跨数千步维持精确计数、自己画地图、记录战斗策略；context 重置后读回自己的笔记继续多小时训练
3. **Sub-agent 架构** —— 子 agent 用干净 context 做深度工作（可能烧几万 token），只回传 1000–2000 token 的蒸馏结论；主 agent 专注综合。见 [[concepts/34-orchestrator-worker-multi-agent]]

选择依据：compaction 适合需要大量来回的对话流；笔记适合有清晰里程碑的迭代开发；多 agent 适合并行探索有回报的研究分析。→ [[concepts/30-compaction-and-note-taking]]

## 面试可直接引用的判断

> 模型越强，越不需要规定性的工程（less prescriptive engineering）；但**把 context 当作珍贵有限资源**这件事不会过时。

这与 [[concepts/08-bitter-lesson]] 一脉相承：补偿模型缺陷的机制会消融，管理稀缺资源的原则会留下。
