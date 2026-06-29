---
title: 源文档 - kube-controller-manager
date: 2026-06-27
tags: [kubernetes, controller-manager, 选主, 驱逐, informer, source]
sources: [kubernetes/components/controller-manager.md]
---

# 源文档摘要：kube-controller-manager

详述 controller-manager 的控制器清单、选主、Informer 与 Node 驱逐。

## 关键内容

- **组成**：kube-controller-manager（内置控制器）+ cloud-controller-manager（启用云时，剥离云逻辑）。
- **控制器清单**：分必启/默认启可选/默认禁三组（Deployment/RS/RC/StatefulSet/DaemonSet/Job/CronJob/Node/HPA/ResourceQuota/Namespace/SA/GC/PV 等）。
- **Leader Election**：`--leader-elect=true`，仅 leader 跑控制器；资源锁从 Endpoint/ConfigMap 迁向 **Lease**。
- **Informer**（v1.7+）：事件通知的只读缓存，大幅减少 apiserver 调用。
- **Node 驱逐**：kubelet 10s 上报、controller 5s 检查；40s→NotReady、5m→驱逐 Pod；按 Zone 限速（`--node-eviction-rate=0.1`，大规模故障降速/停止）。

## 勘误 / 时效性

- ⚠️ EndpointController 在 v1.33+ 弃用（迁 EndpointSlice）；选主 Endpoint 锁已被 Lease 取代（默认 ~v1.20）。
- ⚠️ Metrics 端口 `10252` 明文已约 v1.23 移除（改安全端口）。
- Node 驱逐参数、Zone 限速逻辑仍准确，是理解集群容错的核心。

## 关联

详见实体页 [[entities/kube-controller-manager]] · [[concepts/控制平面与控制循环]] · [[concepts/工作负载控制器]]
