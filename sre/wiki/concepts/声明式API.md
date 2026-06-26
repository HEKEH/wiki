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

## API 对象的三段结构：metadata / spec / status

声明式在对象结构上的直接体现——每个 API 对象都有三类属性：

- **metadata（元数据）**：标识对象。至少含 `namespace`、`name`、`uid`，外加 [[concepts/Label与Selector]] 等。
- **spec（规范）**：用户声明的**期望状态（desired state）**，如"副本数 = 3"。
- **status（状态）**：系统**实际当前状态（actual）**，如"当前副本数 = 2"。

控制器的工作就是不断让 **status 向 spec 收敛**（现 2 个 → 自动再起 1 个）。所以"声明式"落到结构上就是：**用户只写 spec，status 由系统填写**——对应 [[concepts/控制平面与控制循环]] 里 desired/actual 的比对。

> 设计推论：声明式 API 都是**名词性**的（Service、Volume 描述"想要的目标对象"），而非动词性的命令。详见 [[concepts/设计理念]]。

为什么声明式在分布式环境更优：**幂等**。"设副本数为 3"运行多次结果不变；而命令式的"副本数 +1"运行多次结果就错了——这对容易丢操作或重复执行的分布式系统至关重要。
