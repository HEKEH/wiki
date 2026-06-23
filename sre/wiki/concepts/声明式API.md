---
title: 声明式 API
date: 2026-06-23
tags: [声明式, api, 控制器, kubernetes]
sources: [kubernetes/kubernetes简介.md]
---

# 声明式 API（Declarative API）

声明式 API 指用户只描述系统应处于的**期望状态（desired state）**，而非给出达到该状态的**命令式步骤**。系统持续观测实际状态与期望状态的差异，并自动驱动二者收敛。

## 与命令式的对比

- **命令式**：用户逐步下达指令（"创建容器 A、启动 B、再连接 C"），需关心中间状态与执行顺序。
- **声明式**：用户提交期望（"我要 3 个副本在运行"），由控制器负责把现状变成期望，并在偏离时持续纠正。

## 在 Kubernetes 中的体现

[[entities/Kubernetes]] 以声明式 API 为核心：

- 用户向 apiserver 提交期望状态，etcd 持久化之。
- **controller manager** 中的各控制器构成**控制循环（reconcile loop）**，不断比对实际状态与期望状态并采取动作（重建故障 Pod、扩容、滚动更新等）。
- 由此 K8s "消除了编排的需要" —— 用户无需关心中间状态如何转换，系统自动保证应用始终处于期望状态。

这种设计是 K8s 可靠性、弹性与可扩展性的根基，也是 [[concepts/容器编排]] 区别于脚本化运维的关键。
