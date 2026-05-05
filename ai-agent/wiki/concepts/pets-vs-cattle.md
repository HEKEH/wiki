# Pets vs Cattle

tags: [infrastructure, operations, metaphor, scalability]
date: 2026-04-23
sources: [../../raw/Scaling-Managed-Agents-Decoupling.md]
status: active

## Definition

基础设施运维的经典比喻，源自 [Cloud Scaling 博文](https://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/)：

- **Pet** — 有名字的、精心照料的个体，丢失不可接受
- **Cattle** — 可互换的、编号的个体，失败时直接替换

## 在 Agent 架构中的应用

### 耦合设计的问题

将所有 Agent 组件（[Session](session.md)、[Harness](harness.md)、[Sandbox](sandbox.md)）放入单一容器 = 采用了 Pet 模式：

- 容器失败 → session 丢失
- 容器无响应 → 必须"护理"恢复
- 调试需要进入容器 → 容器含用户数据 → 本质上无法调试
- 客户连接 VPC → 必须 peer 网络

### 解耦后的 Cattle 模式

| 组件 | 模式 | 失败处理 |
|------|------|----------|
| Sandbox | Cattle | harness 捕获错误，新 sandbox 通过 `provision()` 创建 |
| Harness | Cattle | 新 harness 通过 `wake(sessionId)` + `getSession()` 恢复 |
| Session | — | 不适用，session 是持久化的日志，不"失败" |

## 更广泛的意义

这是从"不可丢失的单体"到"可替换的组件"的转变。在 Agent 系统中，随着模型能力增长和任务复杂度提升，cattle 模式使系统具备弹性扩展能力——多脑多手的前提正是每个组件都可独立创建和替换。

## 相关概念

- [Harness](harness.md) — cattle 化的 brain
- [Sandbox](sandbox.md) — cattle 化的执行环境
- [Session](session.md) — 持久化的状态，不需要 cattle 化
- [Meta-harness](meta-harness.md) — 设计 cattle 化系统的框架
