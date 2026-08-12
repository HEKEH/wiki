---
title: "How We Built Our Multi-Agent Research System (Anthropic)"
date: 2026-08-11
tags: [source, anthropic, multi-agent, evaluation, production]
sources: [How-We-Built-Our-Multi-Agent-Research-System.md]
status: ingested
---

# How We Built Our Multi-Agent Research System

- 作者：Jeremy Hadfield、[[entities/05-barry-zhang]]、Kenneth Lien、Florian Scholz、Jeremy Fox、Daniel Ford（[[entities/01-anthropic]]）
- 原文：[raw/How-We-Built-Our-Multi-Agent-Research-System.md](../../raw/How-We-Built-Our-Multi-Agent-Research-System.md)

Claude Research 功能的架构复盘。这是**多 agent 系统最具引用价值的一手工程材料**，几乎所有"多 agent 值不值得做"的面试讨论都能用它的数据回答。

## 为什么用多 agent：三个可背的数字

1. **+90.2%**：Claude Opus 4 作 lead + Sonnet 4 作 subagent 的多 agent 系统，在内部 research eval 上比单 agent Opus 4 高 90.2%
2. **token 用量解释 80% 的性能方差**（BrowseComp 评测）；加上工具调用次数与模型选择，三者共解释 95% —— **多 agent 之所以有效，主要是因为它能花掉足够多的 token**
3. **成本**：agent 约为 chat 的 4× token，多 agent 约为 15× → 只有任务价值足够高才经济

不适合多 agent 的场景（原文明确点出）：需要所有 agent 共享同一 context、或 agent 间依赖很多的领域。**大多数编码任务的可并行部分远少于研究任务**，且 LLM 目前不擅长实时协调与委派。适合的是：高价值 + 重并行 + 信息量超单 context + 需对接大量复杂工具。

## 架构：orchestrator-worker

- LeadResearcher 分析查询 → 制定策略 → **把计划写入 Memory 持久化**（因为超过 200k token 会被截断）→ 派生 subagent
- 每个 subagent 独立搜索、用 interleaved thinking 评估工具结果、回传发现
- lead 综合结果，决定是否需要更多研究（可再派 subagent 或调整策略）
- 退出循环后交给 **CitationAgent** 定位引用位置，保证每个论断可溯源
- 与静态 RAG 的对比：**RAG 是一次性取相似 chunk；这里是多步搜索，动态发现、随发现调整**

搜索的本质是**压缩**：subagent 用各自的 context 并行探索，把最重要的 token 蒸馏给 lead。同时提供关注点分离（各自的工具、prompt、探索轨迹），降低路径依赖。

## Prompt 工程八条原则（面试高频）

1. **像 agent 一样思考** —— 用 Console 以真实 prompt/工具模拟，逐步观察，失败模式立刻显形（已经够了还在继续搜、查询过于冗长、选错工具）
2. **教会 orchestrator 如何委派** —— 每个 subagent 需要：目标、输出格式、工具与来源指引、清晰的任务边界。反例："research the semiconductor shortage" 太模糊，导致 3 个 subagent 里 1 个跑去查 2021 汽车芯片危机，另 2 个重复查 2025 供应链
3. **让努力程度匹配查询复杂度** —— 在 prompt 里写死缩放规则：简单事实 1 个 agent + 3–10 次工具调用；直接比较 2–4 个 subagent 各 10–15 次；复杂研究 10+ 个 subagent 且职责明确划分
4. **工具设计与选择至关重要** —— agent-tool 接口和 HCI 一样关键。"在 Slack 里才有的信息，让 agent 去搜网，一开始就注定失败"。给显式启发式：先看全部工具、按用户意图匹配、宽泛外部探索用搜网、优先专用工具
5. **让 agent 自我改进** —— Claude 4 是出色的 prompt 工程师：给它 prompt + 失败模式，它能诊断并改进。他们做了 tool-testing agent，反复调用有缺陷的 MCP 工具并重写其描述，**后续 agent 的任务完成时间下降 40%**
6. **先宽后窄** —— 模仿专家研究员：先短而宽的查询看清全景，再逐步收窄。agent 天然倾向过长过具体的查询（结果很少）
7. **引导思考过程** —— extended thinking 当作可控草稿纸：lead 用来规划（评估工具、判断复杂度、定 subagent 数量与角色）；subagent 用 interleaved thinking 在工具结果后评估质量、找差距、优化下一步查询
8. **并行工具调用改变速度量级** —— 两层并行：lead 一次并行起 3–5 个 subagent；subagent 一次并行用 3+ 工具。复杂查询研究时间**最多降低 90%**

总原则：**灌输好的启发式，而不是刚性规则**；prompt 是"协作框架"（分工、解题方法、努力预算），不是严格指令清单。

## 评估：多 agent 特有的难题

传统 eval 假设"输入 X → 路径 Y → 输出 Z"，但多 agent 系统即使同样起点也会走完全不同的有效路径。方法（→ [[concepts/41-agent-evaluation]]）：

- **立刻开始小样本评估**：早期一个 prompt 改动能把成功率从 30% 拉到 80%，效应量这么大时**约 20 条真实查询就够看出变化**。不要等能做几百条才开始
- **LLM-as-judge 做好了能规模化**：单次 LLM 调用、单个 prompt、输出 0.0–1.0 分 + pass/fail，比多裁判分工更一致。评分维度：事实准确性、引用准确性、完整性、来源质量、工具效率
- **人工评估捕捉自动化遗漏的问题**：人类测试者发现早期 agent 一贯偏好 SEO 内容农场而非权威但排名低的学术 PDF —— 加"来源质量启发式"进 prompt 才解决
- 多 agent 有**涌现行为**：改动 lead agent 会不可预测地改变 subagent 行为，要理解交互模式而非单体行为

## 生产可靠性（[[concepts/43-durable-execution]]）

- **agent 有状态且错误会复合**：不能从头重启（贵且伤体验），要能从出错处**恢复**；同时用模型的智能优雅处理（告诉 agent 工具失败了，它适应得出乎意料地好）+ 确定性保障（重试、定期 checkpoint）
- **调试需要新方法**：非确定性 → 加**全链路 production tracing**，监控 agent 决策模式与交互结构（不看具体会话内容以保护隐私）
- **部署要小心**：agent 是长期运行的有状态网络，用 **rainbow deployment** 渐进切流，避免打断在跑的 agent
- **同步执行是瓶颈**：目前 lead 同步等 subagent 完成，简化了协调但阻塞信息流（lead 无法中途操纵 subagent、subagent 之间无法协调）。异步能带来更多并行，但引入结果协调、状态一致性、错误传播难题

## 附录三条（很多人漏掉，但面试很好用）

1. **end-state evaluation**：对会改变持久状态的多轮 agent，评"最终状态对不对"而非逐轮流程；复杂工作流拆成若干 checkpoint 验状态变化
2. **长程会话管理**：完成阶段后总结并写入外部记忆；接近上限时派生干净 context 的新 subagent，通过精心交接保持连续性；从记忆里取回研究计划而非丢失前功
3. **subagent 直接输出到文件系统**：避免"传话游戏"（game of telephone）。subagent 把成果存到外部系统，只把轻量引用回传给协调者 —— 减少多阶段信息损失和 token 复制开销，特别适合代码、报告、数据可视化这类结构化产出
