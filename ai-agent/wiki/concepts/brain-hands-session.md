# Brain-Hands-Session 解耦模型

tags: [agent-architecture, decoupling, abstraction]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

将 Agent 系统虚拟化为三个可独立替换的组件的架构模型——Brain（大脑）、Hands（双手）、Session（记忆）。这是 [Managed Agents](../entities/Managed-Agents.md) 的核心架构，借鉴了操作系统的抽象思路。

## 三组件映射

| 比喻 | 组件 | 接口 | 职责 | 失败策略 |
|------|------|------|------|----------|
| 🧠 大脑 | [Harness](harness.md) | `wake(sessionId)`, Claude 调用 | 推理 + 路由 | 崩溃后新建，从 session 恢复 |
| ✋ 双手 | [Sandbox](sandbox.md) | `execute(name, input) → string` | 执行 + 操作 | 失败后新建，通过 `provision()` 初始化 |
| 📝 记忆 | [Session](session.md) | `getEvents()`, `emitEvent()` | 持久化日志 | 不适用——session 是持久化的 |

## 演化过程

### v1: 耦合设计（单容器）

```
┌─────────────────────────┐
│     单一容器              │
│  Session + Harness +     │
│  Sandbox + 凭证           │
│         = 🐱 Pet         │
└─────────────────────────┘
```

问题：
- 容器失败 → 一切丢失
- 调试困难（含用户数据）
- 无法连接外部 VPC
- 凭证暴露在不可信代码环境中

### v2: 解耦设计

```
Session (持久化) ←→ Harness (无状态) ←→ Sandbox (可替换)
   📝                 🧠                  ✋
```

优势：
- 每个组件可独立失败和恢复
- [Pets vs Cattle](pets-vs-cattle.md) — 容器和 harness 变为 cattle
- 安全边界清晰——凭证不暴露在 sandbox
- 多脑多手——brain 和 hand 可独立扩展

## 多脑多手

- **多脑**: 多个无状态 harness 可并行运行，按需连接 sandbox；TTFT 大幅降低
- **多手**: 一个 brain 可操作多个 sandbox，brain 之间可传递 hands
- `execute(name, input) → string` 统一接口使 hand 的类型对 harness 透明

## 与操作系统抽象的类比

| OS | Agent |
|----|-------|
| 硬件 → process, file | Agent → session, harness, sandbox |
| `read()` 不关心存储介质 | `execute()` 不关心 sandbox 类型 |
| 抽象数十年稳定 | 接口应比实现更长寿 |

## 相关概念

- [Harness](harness.md) — Brain 的具体实现
- [Sandbox](sandbox.md) — Hands 的具体实现
- [Session](session.md) — Memory 的具体实现
- [Meta-harness](meta-harness.md) — 容纳此解耦模型的系统
- [Pets vs Cattle](pets-vs-cattle.md) — 解耦的运维哲学
