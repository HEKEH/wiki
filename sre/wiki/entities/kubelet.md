---
title: kubelet
date: 2026-06-26
tags: [kubernetes, kubelet, 节点, agent]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes架构原理.md]
---

# kubelet

kubelet 是每个 [[entities/Node]] 上的**主 agent**（每个 Node 一个）。总职责：**接收"本节点该跑哪些 [[entities/Pod]]"，确保这些 Pod 的容器按期望运行，并把现状上报回 apiserver**（见 [[concepts/控制平面与控制循环]] 端到端链路的第 ④⑤ 步）。

## 与分层架构的关系

kubelet **不等于核心层**。[[concepts/分层架构]] 的"核心层"是按功能高度划分的**概念层**，kubelet 是**具体组件**——它实现了核心层"对内提供插件式应用执行环境"的那一块，但核心层更大（还含对外的核心 API）。即：**kubelet 承担核心层的执行环境角色，但核心层 ≠ kubelet**。

## 内部模块 / 职责

| 模块 | 作用 |
| --- | --- |
| **syncLoop（主循环）** | 心脏：不断比对"期望的 Pod"与"实际运行的容器"，驱动收敛 |
| **CRI 客户端** | 调用 container runtime 拉镜像、创建沙箱、起停容器（见 [[concepts/插件机制与可扩展性]]） |
| **PLEG**（Pod Lifecycle Event Generator） | 监听运行时容器状态变化并通知主循环 |
| **Volume Manager** | 挂载/卸载卷，对接 CSI（见 [[concepts/Volume存储]]） |
| **网络（CNI）** | 起 Pod 时配置网络、分配 IP（见 [[concepts/Pod网络模型]]） |
| **Probe Manager** | 执行 liveness/readiness/startup 探针（见 [[concepts/健康检查]]） |
| **Image Manager** | 镜像拉取与垃圾回收 |
| **cAdvisor** | 采集容器 CPU/内存等资源指标 |
| **Container/cgroup 管理** | 按 [[concepts/资源限制]] 通过 cgroups 约束资源、管理 QoS |
| **Eviction Manager** | 节点资源紧张时驱逐 Pod |
| **Device Manager** | 接入 GPU 等设备插件 |
| **Node 状态/心跳** | 注册节点、定期上报 NodeStatus / node lease |

## 作为接口的调用方

kubelet 是 **CRI / CNI / CSI 的客户端（调用方）**——它调用这些接口让容器跑起来、联网、挂卷；具体实现（containerd、Calico、CSI driver）由生态系统层的插件提供（见 [[concepts/插件机制与可扩展性]]）。

> 注意：kubelet 与 [[entities/kubectl]] 是完全不同的东西——kubelet 是节点 agent，kubectl 是命令行工具。

## 静态 Pod 与自举

**manifest** 指描述 K8s 对象期望状态的 YAML/JSON 文件（即 [[concepts/声明式API]] 的输入）。普通 manifest 由人/工具写好、经 `kubectl apply` 交给 apiserver 生效；而 kubelet 还能**直接管理"静态 Pod（static Pod）"**——读取**节点本地目录**（由 kubelet 配置 `staticPodPath` 指定，kubeadm 默认 `/etc/kubernetes/manifests/`）里的 manifest 起 Pod，**完全不经过 apiserver、不需要 scheduler**。

| | 普通 manifest | 本地/静态 manifest |
| --- | --- | --- |
| 来源 | 人 / Helm / Kustomize 编写 | 引导工具（kubeadm）**生成** |
| 位置 | 你的仓库 → 提交集群 | 节点 `/etc/kubernetes/manifests/` |
| 生效 | `kubectl apply` → apiserver | kubelet **直接读目录** |

这是**自举控制平面**的方式：`kubeadm init` 自动生成 apiserver / controller-manager / scheduler / [[entities/etcd]] 的静态 Pod manifest 写入该目录，kubelet 一启动就把它们跑起来——解决"apiserver 还没起、谁来起 apiserver"的循环依赖（见 [[concepts/设计理念]] 引导原则）。

> 静态 Pod 起来后，kubelet 会在 apiserver 里建一个**只读的"镜像 Pod（mirror pod）"**，所以 `kubectl get pods` 也能看到它们，但它们由 kubelet 管、非 scheduler 调度。

## 相关

- 所在节点与同级组件：[[entities/Node]]（kubelet / container runtime / kube-proxy）
- 控制平面协作：[[concepts/控制平面与控制循环]]
