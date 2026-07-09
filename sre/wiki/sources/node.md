---
title: Node（资源对象）
date: 2026-06-30
tags: [kubernetes, node, 节点, taint, cordon, 优雅关闭]
sources: [kubernetes/objects/node.md]
---

# 源摘要：Node

feisky handbook `/concepts/objects/node.md` 的要点（深入页见 [[entities/Node]]）。

## 核心

- Node 是 Pod 运行的主机；**K8s 不创建 Node，只管理其上资源**（kubelet 启动时自注册）。每节点至少跑 container runtime + kubelet + kube-proxy。
- **Node Controller**（在 [[entities/kube-controller-manager]] 内）：维护 Node 状态、与 cloud provider 同步、分配容器 CIDR、删除带 NoExecute taint 节点上的 Pod。

## Node 状态

- 地址（hostname/内外网 IP）、**Condition**（Ready / MemoryPressure / DiskPressure / PIDPressure）、Capacity、Allocatable、Info。

## 运维相关

- **Taint/Toleration**：`kubectl taint nodes ...` 排斥 Pod（机制见 [[entities/kube-scheduler]]）。
- **维护**：`kubectl cordon`（标记不可调度但不动现有 Pod）/ `drain`（驱逐后维护，配合 PDB）。
- **优雅关闭**：`ShutdownGracePeriod` + `ShutdownGracePeriodCriticalPods`（默认 0=关闭），关机时先停普通 Pod、最后停关键 Pod。
- **非优雅关闭**：手动打 `node.kubernetes.io/out-of-service` taint，让卡在宕机节点上的 StatefulSet Pod/卷能在别处快速恢复。

## 勘误 / staleness

- ⚠️ 示例提到 `rkt` 运行时已废弃；Condition 中的 **OutOfDisk 已移除**（并入 DiskPressure）；非优雅关闭 `NodeOutOfServiceVolumeDetach` 已 GA。
