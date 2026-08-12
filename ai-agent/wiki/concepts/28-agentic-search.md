---
title: Agentic Search 与 Just-in-Time Context
date: 2026-08-11
tags: [context-engineering, retrieval, agentic-search, progressive-disclosure]
sources: [Effective-Context-Engineering-for-AI-Agents.md, How-We-Built-Our-Multi-Agent-Research-System.md]
---

# Agentic Search 与 Just-in-Time Context

**2025–2026 年 agent 架构最重要的一次范式转移**：从"推理前把资料检索好塞进去"转向"给 agent 工具，让它自己边走边取"。

## 两种取上下文的方式

| | 预检索（pre-inference retrieval） | Just-in-time（agentic search） |
|---|---|---|
| 时机 | 推理开始前一次性完成 | 运行时按需，多轮 |
| 存什么 | 向量化后的内容本体 | **轻量标识符**：文件路径、查询语句、URL |
| 取什么 | top-k 相似片段 | agent 自己决定读哪个、读多少 |
| 优点 | 快、可预测 | 不受索引过期影响、能多跳、能判断来源质量 |
| 缺点 | 相似 ≠ 相关；索引会过期；无法多跳 | 慢；需要好工具和启发式，否则绕远路 |

Claude Code 的例子：面对大型数据库，它写定向查询、存结果、用 `head`/`tail` 分析，**从不把完整数据载入 context**。

## 为什么它有效：元数据即信号

- `tests/test_utils.py` 与 `src/core_logic/test_utils.py` 同名但含义完全不同
- 文件大小暗示复杂度、命名约定暗示用途、时间戳可作相关性代理
- 文件夹层级本身就是一种人类维护了几十年的**组织与索引系统**

这类比人的认知：我们不背下整个语料，而是建立文件系统、收件箱、书签，按需取用。

## Progressive Disclosure（渐进披露）

让 agent 通过探索**逐层**发现上下文：每次交互产生的信息决定下一步决策，只在工作记忆里保留必需内容。同一原则的三处应用：

1. **数据**：先列目录再读文件，而非一次读完
2. **工具**：文件树 / `search_tools`（带详略级别）按需加载工具定义 → [[concepts/32-progressive-disclosure-and-code-execution]]
3. **搜索策略**：**先宽后窄** —— 短而宽的查询看全景，再逐步收窄（agent 天然倾向过长过具体的查询，结果反而少）

## 混合策略（生产上的实际选择）

> Claude Code 采用混合模型：`CLAUDE.md` 直接放进 context，同时用 glob/grep 即时检索，绕开索引过期与语法树复杂度问题。

判断边界：**内容越动态、越大、越难预判需要哪部分 → 越偏 agentic**；内容稳定、延迟敏感 → 越偏预检索。法律、金融这类相对静态的领域更适合混合。

## 代价与前提

- 运行时探索比预计算慢（延迟 + token）
- 需要"opinionated 且用心"的工程：工具与启发式要足够好，否则 agent 会误用工具、走死路、错过关键信息
- 因此它与 [[concepts/25-tool-use-and-function-calling]] 的质量强耦合：**agentic search 的上限由工具质量决定**

## 面试怎么讲

被问"你们的 RAG 怎么做的"时，一个高分回答结构：

1. 先说清标准管线与你们的指标（说明基本功，见 [[concepts/27-rag]]）
2. 指出静态检索的局限（相似≠相关、无法多跳、索引过期）
3. 给出 agentic search / just-in-time 的替代方案与它的代价
4. 落到**混合策略**与选择判据 —— 这一步最能体现工程判断力

## 与其他概念的关系

- [[concepts/09-context-engineering]]：本页是其"运行时取 context"章节的展开
- [[concepts/29-context-rot-and-attention-budget]]：解释了为什么"少而准"胜过"多而全"
- [[concepts/34-orchestrator-worker-multi-agent]]：subagent 是 agentic search 的并行化形态

## 来源

- [[sources/06-effective-context-engineering]]、[[sources/07-multi-agent-research-system]]
