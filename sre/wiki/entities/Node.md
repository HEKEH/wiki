---
title: Node
date: 2026-06-23
tags: [kubernetes, node, 服务节点, taint, cordon, 优雅关闭]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes集群.md, kubernetes/objects/node.md]
---

# Node

Node 是 [[entities/Pod]] 真正运行的主机，可以是物理机也可以是虚拟机。在集群中扮演**服务节点**角色——真正运行容器、管理镜像，并提供集群内的服务发现与负载均衡。

## 必备组件

为了管理 Pod，每个 Node 至少要运行：

- **container runtime**（如 docker、rkt，现主流为 containerd）—— 实际运行容器。
- **[[entities/kubelet]]** —— 维护容器生命周期，管理 Volume（CVI）与网络（CNI）。
- **kube-proxy** —— 为 [[entities/Service]] 提供集群内的服务发现与负载均衡。

## 在集群中的位置

一个 Kubernetes 集群由**控制节点（controller）**、**服务节点（Node）**和**分布式存储 etcd** 组成。Node 即服务节点。详见 [[concepts/集群架构]]。

## Node 不是 K8s 创建的

与 Pod/Namespace 不同，**K8s 不创建 Node，只管理其上资源**——kubelet 启动时向控制面**自注册** Node 对象。手写 Node manifest 也只是声明"应该有这台机"，查不到真机就不会往上调度。维护它的是 **Node Controller**（在 [[entities/kube-controller-manager]] 内）：维护 Node 状态、与 cloud provider 同步、分配容器 CIDR、删除带 NoExecute taint 节点上的 Pod。

## Node 状态

- **地址**（hostname/内外网 IP）、**Condition**（`Ready` / `MemoryPressure` / `DiskPressure` / `PIDPressure`）、**Capacity**（总资源）、**Allocatable**（可分配给 Pod 的量）、**Info**（内核/运行时/OS 版本）。
- Condition 由 [[entities/kubelet]] 上报，也驱动 [[entities/kube-scheduler]] 的调度决策（如 DiskPressure 拒新 Pod）。

## 运维操作

- **Taint/Toleration**：`kubectl taint nodes ...` 让节点排斥不容忍它的 Pod（机制见 [[entities/kube-scheduler]]）。
- **维护**：`kubectl cordon`（标记不可调度、不动现有 Pod）→ `kubectl drain`（驱逐 Pod 后再维护，受 [[entities/Pod]] 的 PodDisruptionBudget 约束）。
- **优雅关闭**：kubelet 配 `ShutdownGracePeriod` + `ShutdownGracePeriodCriticalPods`（默认 0=关闭），关机时先停普通 Pod、最后停关键 Pod。
- **非优雅关闭**：节点突然宕机时 kubelet 来不及处理；手动打 `node.kubernetes.io/out-of-service` taint，让卡住的 [[entities/StatefulSet]] Pod 和卷能在别处快速恢复。

> ⚠️ 源文档提到的 `rkt` 运行时已废弃；Condition 里的 **OutOfDisk 已移除**（并入 DiskPressure）。

## 备注

部分资源**不属于任何 namespace**，Node 即是其中之一（另有 persistentVolumes 等）。见 [[entities/Namespace]]。
