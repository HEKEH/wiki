# Initializer/Coding Agent 模式

tags: [agent-architecture, harness-design, multi-session]
date: 2026-04-24
sources: [../../raw/Effective-harnesses-for-long.md]
status: active

## Definition

一种双 Agent 分工模式，用于解决 [Long-Running Agent](long-running-agent.md) 跨 context window 工作时的状态衔接问题。第一个 session 使用专门的 Initializer Agent 搭建环境，后续所有 session 使用 Coding Agent 做增量推进。

> 注：这两个 "Agent" 仅拥有不同的初始 user prompt，system prompt、工具集和整体 harness 完全相同。

## Initializer Agent

仅在第一个 session 运行，职责是搭建后续所有 session 依赖的环境基础设施：

| 产出 | 作用 |
|---|---|
| `init.sh` | 环境初始化脚本（启动开发服务器、安装依赖等），消除每个 session 花时间搞清"如何运行" |
| `feature_list.json` | 细粒度 feature 列表（见 [Feature List Pattern](feature-list-pattern.md)），为"完成"提供客观标准 |
| `claude-progress.txt` | 追加写入的交接笔记，记录每个 session 的进展与下一步（详见下文） |
| 初始 git commit | 展示哪些文件被添加，提供版本回滚基线 |

## Coding Agent

后续所有 session 使用，标准化的启动流程：

```
pwd → 读 claude-progress.txt → 读 feature_list.json → git log
→ 运行 init.sh → 冒烟测试（验证已有功能正常）
→ 选一个未完成 feature → 实现 → 端到端测试 → commit → 写 progress
```

### 关键行为约束

- **一次只做一个 feature** — 解决 one-shotting 倾向
- **结束前 commit + 写 progress** — 留下结构化的交接信息
- **先验证再推进** — 发现已有 bug 立即修复，避免在坏基础上叠加新代码
- **端到端测试** — 用浏览器自动化工具（如 Puppeteer MCP）像人类用户一样测试，而非仅单元测试

## claude-progress.txt：轮班交接笔记

`claude-progress.txt` 是自由文本的追加写入日志，由 Initializer Agent 创建，每个 Coding Agent session 结束前必须更新。它是让零记忆的下一个 Agent 快速恢复工作状态的关键工件。

### 每个 session 必须写入的内容

根据 [配套 quickstart 仓库](https://github.com/anthropics/claude-quickstarts/blob/main/autonomous-coding/prompts/coding_prompt.md) 的 Step 9 指令：

1. **What you accomplished this session** — 本 session 做了什么
2. **Which test(s) you completed** — 完成了哪些 feature 测试
3. **Any issues discovered or fixed** — 发现或修复了什么问题
4. **What should be worked on next** — 下一步该做什么
5. **Current completion status** — 当前进度（如 "45/200 tests passing"）

### 示例

```
## Session 1 — Initializer

- Read app_spec.txt and created feature_list.json with 200+ features
- Created init.sh for development environment setup
- Initialized git repo with initial commit
- Set up basic project structure (frontend + backend directories)
- Began implementing: basic chat interface
- Status: 0/200 tests passing
- Next: implement fundamental chat features

## Session 2 — Coding Agent

- Implemented: New chat button creates a fresh conversation
- Implemented: User can type a query and press enter to send
- Fixed: development server not starting on port 3000
- Fixed: white-on-white text in sidebar (contrast issue)
- Status: 3/200 tests passing
- Next: continue with chat response display features
```

### 与 Feature List 的互补关系

| | `claude-progress.txt` | `feature_list.json` |
|---|---|---|
| 性质 | 主观的上下文叙述 | 客观的状态记录 |
| 格式 | 自由文本，追加写入 | 结构化 JSON，仅改 `passes` 字段 |
| 回答的问题 | 做了什么、遇到什么、下一步是什么 | 哪些 feature 通过、哪些未通过 |
| 类比 | 轮班工程师的交接笔记 | 质检报告的 checklist |

两者共同构成跨 session 的双重状态传递：progress file 提供**叙事上下文**，feature list 提供**可验证的客观标准**。

## 设计原则

1. **让 Agent 快速理解工作状态** — progress file + git history + feature list 三重上下文
2. **保持环境干净** — 每次离开时代码应处于可合并到 main 的状态
3. **提供客观完成标准** — feature list 的 `passes` 字段防止 premature victory
4. **标准化启动流程** — 减少每个 session 的探索开销

## 与其他架构的关系

- 是 [Harness](harness.md) 的一种具体实现策略——在同一个 harness 内通过不同 prompt 实现角色分工
- 与 [Meta-harness](meta-harness.md) 不同——meta-harness 容纳不同 harness 实现，本模式在同一 harness 内切换 prompt
- 是 [Session](session.md) 跨窗口衔接的应用模式

## 局限与开放问题

- 当前优化对象是全栈 Web 应用开发，泛化能力待验证
- 单一通用 coding agent vs 多 agent 架构（专门的测试 agent、QA agent）哪个更优尚未定论
- 强依赖结构化文件（JSON progress、feature list），非结构化领域可能需要不同的状态传递方式

## 相关概念

- [Long-Running Agent](long-running-agent.md) — 本模式解决的问题域
- [Feature List Pattern](feature-list-pattern.md) — Initializer Agent 创建、Coding Agent 消费的核心工件
- [Harness](harness.md) — 承载此模式的编排循环
- [Session](session.md) — 跨 context window 的持久化存储
- [Context Engineering](context-engineering.md) — 管理 context window 的工程实践
