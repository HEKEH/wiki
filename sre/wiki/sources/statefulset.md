---
title: StatefulSet（资源对象）
date: 2026-06-30
tags: [kubernetes, statefulset, 有状态, headless-service, pvc]
sources: [kubernetes/objects/statefulset.md]
---

# 源摘要：StatefulSet

feisky handbook `/concepts/objects/statefulset.md` 的要点（深入页见 [[entities/StatefulSet]]）。

## 核心：为有状态服务提供三种"稳定"

1. **稳定持久存储**：经 `volumeClaimTemplates` 为每个 Pod 各创建一份 PVC，重建后挂回原盘。
2. **稳定网络标识**：PodName/HostName 不变，靠 **Headless Service（`clusterIP: None`）** 提供固定 DNS。
3. **有序部署/扩展（0→N-1）、有序收缩/删除（N-1→0）**。

## 组成与 DNS

- 三件套：定义 DNS 域的 **Headless Service** + **volumeClaimTemplates** + StatefulSet 本体。
- Pod DNS 格式：`<statefulSetName>-{0..N-1}.<serviceName>.<namespace>.svc.cluster.local`。

## 更新与管理策略

- `.spec.updateStrategy`：**RollingUpdate**（按逆序滚动）/ **OnDelete**（删旧 Pod 才建新）。
- **Partitions**：`rollingUpdate.partition=K` → 仅序号 ≥ K 的 Pod 更新（灰度）。
- `.spec.podManagementPolicy`：**OrderedReady**（默认，按序等 Ready）/ **Parallel**（并行起停）。

## 注意事项

- 删除 StatefulSet **不删 PVC**（保数据），需手动清理。
- Headless Service 须在 StatefulSet 之前创建。

## 勘误 / staleness

- ⚠️ 源称"OnDelete 是默认更新策略"——已过时：**apps/v1 默认是 RollingUpdate**（OnDelete 为兼容 v1.6 的旧默认）。
- ⚠️ 示例用 `k8s.gcr.io`、`apps/v1beta1`、nslookup 显示 `kube-dns`——现为 **registry.k8s.io / apps/v1 / CoreDNS**。
- StatefulSet 自 **v1.9 GA**（源"推荐 v1.9+"仍成立）。
