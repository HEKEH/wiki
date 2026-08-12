---
title: OpenAI
date: 2026-08-11
tags: [company, openai, entity]
sources: [OpenAI-A-Practical-Guide-to-Building-Agents.md, OpenAI-Agents-SDK-Documentation.md]
---

# OpenAI

AI 研究与部署公司。在本知识库中作为**与 Anthropic 并列的两大 agent 方法论来源**出现。

## 与 Agent 相关的产出

| 类型 | 内容 |
|---|---|
| 方法论 | [[sources/05-openai-practical-guide-to-building-agents]]（34 页官方白皮书） |
| 框架 | [[entities/11-openai-agents-sdk]]（code-first，非声明式） |
| 模型能力 | function calling、structured outputs、Responses API、computer-use 模型、推理模型 |
| 平台能力 | Conversations API、hosted tools（web search / file search / code interpreter / container shell + skills）、moderation API |

## 立场特征（与 Anthropic 对照）

- **单 agent 优先**：先把单 agent 能力做满，再拆多 agent；判据是"复杂逻辑"与"工具重叠度"
- **两种多 agent 形态**：manager（agents as tools）与 decentralized（handoffs），见 [[concepts/35-manager-vs-decentralized]]
- **明确批评声明式图框架**：要求预先定义所有分支/循环会在动态工作流下笨重，还要学 DSL；Agents SDK 选 code-first
- **护栏当一等概念**：七类护栏 + 乐观执行 + tripwire，见 [[concepts/37-guardrails]]
- **模型选择方法论**：先建 eval 基线 → 用最强模型达标 → 再换小模型压成本

## 与 Anthropic 观点的异同

- **相同**：简单优先；工具设计是核心；护栏与人工介入必需；单 agent → 多 agent 的渐进路径
- **不同**：Anthropic 更强调 context engineering 与长程/子 agent 架构（[[concepts/30-compaction-and-note-taking]]），OpenAI 更强调编排模式与产品化护栏清单

对比详见 [[analysis/03-framework-comparison]]。
