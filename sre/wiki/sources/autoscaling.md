---
title: Autoscaling / HPA（资源对象）
date: 2026-06-30
tags: [kubernetes, hpa, vpa, 自动扩缩容, metrics-server]
sources: [kubernetes/objects/autoscaling.md]
---

# 源摘要：Autoscaling（HPA）

feisky handbook `/concepts/objects/autoscaling.md` 的要点（深入页见 [[entities/Autoscaling]]）。

## 核心

- **HPA（Horizontal Pod Autoscaler）**：按 CPU 利用率或自定义 metrics 自动调整副本数（作用于 Deployment/RS/RC）。
- controller-manager 每 ~15s（`--horizontal-pod-autoscaler-sync-period`）查指标算目标副本数。
- 三种 metrics：预定义（利用率，如 CPU）/ 自定义 Pod metrics（原始值）/ object metrics；支持多指标。
- **前提：必须部署 metrics-server**；Node 扩缩容是另一回事（Cluster Autoscaler）。

## 用法

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

- 状态条件：AbleToScale / ScalingActive / ScalingLimited。

## VPA（Vertical Pod Autoscaler，独立项目）

- 按历史/当前用量自动调 Pod 的 CPU/内存 **请求**；`updateMode`：Off（仅推荐）/ Initial（仅创建时）/ Recreate（驱逐重建套用）/ Auto（≈Recreate）。VPA 1.4+ 新增 **`InPlaceOrRecreate`**（alpha，依赖 K8s 1.33 原地 resize）：优先原地改、不行则**回退重建**——非"永不重启"。
- 不要在同一资源上同时用 HPA 和 VPA。

## v1.33：可配置容忍度

- `behavior.scaleUp/scaleDown.tolerance` 分别设扩/缩容触发阈值（取代固定 10%），`HPAConfigurableTolerance` 门控（Alpha）。

## 勘误 / staleness

- ⚠️ API：`autoscaling/v2beta1` 已废弃——现为 **`autoscaling/v2`（GA 自 v1.23）**，schema 用 `target.type/averageUtilization`（非 `targetAverageUtilization`）。
- ⚠️ **Heapster 早已移除**，指标来源改 metrics-server / custom-metrics API；示例 `k8s.gcr.io`→`registry.k8s.io`。
