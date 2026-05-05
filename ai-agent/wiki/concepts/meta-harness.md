# Meta-harness

tags: [agent-architecture, abstraction, design-philosophy]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

一种不限定特定 harness 实现的系统设计——对接口有主见，对实现无主见。[Managed Agents](../entities/Managed-Agents.md) 即是一个 meta-harness。

## 设计哲学

### 核心原则：为"尚未设想的程序"设计系统

借鉴操作系统的设计智慧：OS 通过将硬件虚拟化为抽象（`process`、`file`）来支持未来未知的程序。`read()` 不关心它访问的是 1970 年代的磁盘还是现代 SSD——抽象层保持稳定，底层实现自由变化。

Meta-harness 将同样的模式应用于 Agent：

| OS 抽象 | Agent 抽象 |
|---------|-----------|
| Process | [Session](session.md) |
| File | Event |
| `read()` | `getEvents()` |
| Hardware → Abstraction | Agent Components → Interfaces |

### 接口优于实现

- **有主见的**：Claude 需要[操纵状态](session.md)的能力、[执行计算](sandbox.md)的能力、多脑多手的扩展能力
- **无主见的**：具体有多少 brain 和 hand、它们在哪里运行、使用什么特定 harness

## 可容纳的 Harness 类型

- **Claude Code** — 通用型优秀 harness
- **任务特定 harness** — 窄域内表现卓越的专用 harness
- **未来 harness** — 随模型能力进化而设计的尚未设想的 harness

## 与 The Bitter Lesson 的共鸣

[The Bitter Lesson](bitter-lesson.md) 告诉我们：利用算力的通用方法最终胜过人类设计的特定方法。Meta-harness 延续了这一洞见——不为特定模型缺陷设计特定方案，而是提供足够通用的接口让实现自由进化。

## 相关概念

- [Harness](harness.md) — meta-harness 所容纳的实现
- [Session](session.md) — meta-harness 的核心抽象之一
- [Sandbox](sandbox.md) — meta-harness 的核心抽象之一
- [The Bitter Lesson](bitter-lesson.md) — 哲学共鸣
- [Pets vs Cattle](pets-vs-cattle.md) — meta-harness 所采用的运维模式
