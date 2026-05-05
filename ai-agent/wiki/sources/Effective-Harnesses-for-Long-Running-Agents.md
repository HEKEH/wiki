# Effective Harnesses for Long-Running Agents

tags: [agent-architecture, harness-design, long-running]
date: 2026-04-24
sources: [../../raw/Effective-harnesses-for-long.md]
author: Justin Young
status: active

## 概述

Anthropic 工程博客文章，探讨如何让 Agent 在跨越多个 context window 的长时运行任务中持续有效工作。作者 Justin Young，基于 Claude Agent SDK 的内部实验，提出 **Initializer Agent + Coding Agent** 的双 Agent 设计模式。

**原文链接**: [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

**配套代码**: [claude-quickstarts/autonomous-coding](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding)

## 核心问题

长时运行 Agent 面临两个经典失败模式：

1. **One-shotting** — Agent 试图一次性完成所有工作，context 耗尽时留下半成品代码，下一个 session 无法理解之前的状态
2. **Premature Victory** — Agent 看到已有进展就宣布项目完成

即使有 compaction（上下文压缩），Opus 4.5 在循环中也无法仅凭高层提示完成生产级 Web 应用。

## 解决方案：双 Agent 设计

### Initializer Agent（仅第一个 session）

- 创建 `init.sh` 环境初始化脚本（启动开发服务器、安装依赖等，消除后续 session 的启动探索成本）
- 创建 `claude-progress.txt` 交接笔记（追加写入的自由文本日志，每个 session 记录：做了什么、完成了哪些测试、发现/修复了什么问题、下一步该做什么、当前进度）
- 创建 `feature_list.json` 细粒度 feature 列表（200+ 项，每项有 `passes` 字段）
- 初始 git commit

### Coding Agent（所有后续 session）

启动流程：pwd → 读 progress → 读 feature list → git log → 运行 init.sh → 冒烟测试 → 选一个 feature → 实现 → 测试 → commit → 写 progress

## 关键机制

| 机制 | 作用 | 要点 |
|---|---|---|
| Feature List (JSON) | 拆解需求为细粒度可验证项 | 用 JSON 而非 Markdown，防止模型擅自修改；用"strongly-worded"指令禁止删除测试 |
| Incremental Progress | 一次只做一个 feature | 每次 commit + 写 progress summary；可用 git revert 回滚 |
| End-to-End Testing | Puppeteer MCP 浏览器自动化测试 | 而非仅单元测试；大幅减少"自以为完成"的错误 |
| Session 启动流程 | 标准化上下文恢复 | 先验证已有功能正常，再开始新 feature |

## 失败模式与解决方案对照

| 问题 | Initializer Agent | Coding Agent |
|---|---|---|
| 宣布胜利太早 | 创建 feature list | 每个 session 读 feature list，选一个未完成项 |
| 留下 bug 或未记录的进度 | 初始 git repo + progress notes | 启动时读 progress + git log + 冒烟测试；结束时 commit + 写 progress |
| 过早标记 feature 完成 | 创建 feature list | 自行验证所有 feature，仅在充分测试后标记 passes |
| 花时间搞清如何运行 | 写 init.sh（环境初始化脚本） | 启动时执行 init.sh，快速进入可用状态 |

## 开放问题

- 单一通用 coding agent vs 多 agent 架构（测试 agent、QA agent、代码清理 agent）
- 这些实践能否泛化到 Web 开发以外的领域（科研、金融建模等）

## 与其他来源的关系

- 延伸了 [Scaling Managed Agents: Decoupling](Scaling-Managed-Agents-Decoupling.md) 中的 [Harness](../concepts/harness.md) 概念——本文是 harness 的具体实现策略
- 与 [Session](../concepts/session.md) 互补——解决了跨 Session 的状态衔接问题
- 与 [Context Anxiety](../concepts/context-anxiety.md) 相关——是 context window 限制的工程应对

## 相关概念

- [Long-Running Agent](../concepts/long-running-agent.md) — 跨多个 context window 工作的 Agent
- [Initializer/Coding Agent 模式](../concepts/initializer-coding-agent.md) — 双 Agent 分工设计
- [Feature List Pattern](../concepts/feature-list-pattern.md) — 用结构化 JSON 拆解和追踪需求
- [Harness](../concepts/harness.md) — Agent 的编排循环
- [Session](../concepts/session.md) — 跨 context window 的持久化存储
- [Context Engineering](../concepts/context-engineering.md) — 管理 context window 的工程实践
