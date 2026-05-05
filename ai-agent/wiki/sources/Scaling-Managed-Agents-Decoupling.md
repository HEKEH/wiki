# Scaling Managed Agents: Decoupling the Brain, Hands, and Session

tags: [agent-architecture, decoupling, harness-design, anthropic, infrastructure]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: ingested

## Summary

Anthropic 工程博客文章，阐述 [Managed Agents](../entities/Managed-Agents.md) 的架构设计哲学：借鉴操作系统的抽象思路（`process`、`file`），将 Agent 的三大组件——[Session](../concepts/session.md)、[Harness](../concepts/harness.md)、[Sandbox](../concepts/sandbox.md)——解耦为可独立替换的接口，使系统足以容纳"尚未设想的程序"。

## 核心论点

1. **Harness 假设会过时** — Claude Sonnet 4.5 存在 [Context Anxiety](../concepts/context-anxiety.md)，需要 harness 添加上下文重置；但 Opus 4.5 消除了该行为，重置逻辑变成死代码。Harness 必须持续进化。
2. **解耦 > 耦合** — 将所有组件放入单一容器 = 采用了 [Pets vs Cattle](../concepts/pets-vs-cattle.md) 中的"宠物"模式；解耦后每个组件变为可替换的 cattle。
3. **安全边界不可妥协** — 凭证永远不应暴露在 sandbox 中；通过 Git remote 注入和 MCP 代理+保险库模式实现。
4. **Session 是持久化的上下文对象** — [Session](../concepts/session.md) ≠ Context Window；session 保证数据不丢，harness 负责上下文工程（compaction/trimming），二者关注点分离。
5. **[Meta-harness](../concepts/meta-harness.md) 哲学** — 对接口有主见，对实现无主见；接口应比实现更长寿。

## 三大组件

| 组件 | 比喻 | 核心接口 | 职责 |
|------|------|----------|------|
| Session | 记忆 | `getSession()`, `emitEvent()`, `getEvents()` | 追加写入的事件日志，持久化存储 |
| Harness | 大脑 | `wake(sessionId)`, 调用 Claude + 路由工具 | 无状态循环，崩溃可重启，可替换 |
| Sandbox | 双手 | `execute(name, input) → string`, `provision({resources})` | 执行环境，容器化，可替换 |

## 关键设计决策

### 不要养宠物

单容器耦合意味着：容器失败 → session 丢失；容器无响应 → 只能"护理"；调试需要进入包含用户数据的容器 → 本质上无法调试。解耦后：容器死亡 → harness 捕获工具调用错误 → Claude 可重试 → 新容器通过 `provision()` 初始化。

### Harness 也是 Cattle

Harness 崩溃不影响 session——新 harness 通过 `wake(sessionId)` 恢复，用 `getSession()` 取回事件日志，从最后事件继续。

### 安全边界

- **Git**: 用 repo 的 access token 在 sandbox 初始化时 clone 并注入 git remote，sandbox 内 `push`/`pull` 无需 token
- **MCP**: OAuth token 存在安全保险库中，Claude 通过专用代理调用 MCP 工具，代理从保险库取凭证，harness 始终不接触凭证

### 多脑多手

- **多脑**: 解耦后 TTFT p50 降低 ~60%，p95 降低 >90%。无需 sandbox 的 session 不再等待容器启动。
- **多手**: `execute(name, input) → string` 统一接口支持容器、手机、任意工具。Brain 可将 hands 传递给其他 brain。

## 与其他工作的关联

- 延续 [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) 和 [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- 上下文工程部分关联 [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- 借鉴 [The Bitter Lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)（通用方法胜过特定设计）和 [The Art of Unix Programming](http://www.catb.org/esr/writings/taoup/html/ch03s01.html)（为尚未设想的程序设计系统）

## 作者

Lance Martin, Gabe Cemaj, Michael Cohen
