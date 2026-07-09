---
title: Autoscaling (HPA / VPA)
date: 2026-06-30
tags: [kubernetes, hpa, vpa, 自动扩缩容, metrics-server]
sources: [kubernetes/objects/autoscaling.md]
---

# Autoscaling (HPA / VPA)

自动伸缩有三个层次，别混淆：**HPA**（改副本数）、**VPA**（改单 Pod 资源）、**Cluster Autoscaler**（改节点数）。本页讲前两者；手动扩缩容见 [[concepts/扩缩容与滚动升级]]。

## HPA（Horizontal Pod Autoscaler）：自动调副本数

- 按 **CPU 利用率 / 内存 / 自定义 metrics** 自动增减 [[entities/Deployment]]/[[entities/ReplicaSet]] 的副本数。
- 由 [[entities/kube-controller-manager]] 每 ~15s 拉指标、算目标副本数（`期望副本 = 当前副本 × 当前指标/目标指标`）。
- **目标值（如 CPU 50%）是"每个 Pod 的平均利用率"**：总负载基本固定、摊给 N 个 Pod，Pod 越多每个越闲。所以**指标高于目标 → 加 Pod 摊薄负载、把平均利用率降回目标；低于目标 → 减 Pod**（方向别搞反：忙不过来是 Pod *太少*，减 Pod 只会让剩下的更忙）。例：5 副本、当前 60%、目标 50% → `5×60/50=6`，扩到 6 个。
- **前提：集群装了 metrics-server**（CPU/内存来自它；自定义指标走 custom-metrics API）。

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

声明式等价（推荐，可入库/审查/GitOps）——HPA 是**独立对象**，经 `scaleTargetRef` 按"名字+类型+同 namespace"锁定目标：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache            # HPA 自己的名字
spec:
  scaleTargetRef:             # 指向要伸缩的 Deployment（非 label 匹配）
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1              # --min
  maxReplicas: 10             # --max
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # --cpu-percent
```

> ⚠️ 用了 HPA 就**别在 Deployment 清单里写死 `spec.replicas`**：副本数是同一字段，带 `replicas` 反复 `apply`（或 GitOps 同步）会与 HPA 来回覆盖、**抖动**。删掉该字段后 `apply` 不再碰它，副本数全交给 HPA。

- 三种指标类型：Resource（利用率）/ Pods（原始值）/ Object；可组合多指标取最大需求。
- 状态条件：`AbleToScale` / `ScalingActive` / `ScalingLimited`。
- **v1.33**：可配置容忍度 `behavior.scaleUp/scaleDown.tolerance`（取代固定 10% 死区，扩容更灵敏、缩容更稳）。

## VPA（Vertical Pod Autoscaler）：自动调单 Pod 资源

- 据历史/实时用量自动调容器的 CPU/内存 **requests**（独立项目，非核心组件）。`updateMode`：**Off**（仅出推荐值，最安全的起步方式）/ **Initial**（仅 Pod 创建时设一次）/ **Recreate**（总是**驱逐重建** Pod 来套用新值）/ **Auto**（当前 ≈ Recreate）。
- **原地更新** `InPlaceOrRecreate`（VPA 1.4+ alpha，底层依赖 K8s 1.33 的原地 resize，见 [[entities/Pod]]）：**优先原地改、不重启**；但原地不可行时（本节点放不下需换机、缩内存 limit、会跨 [[concepts/资源限制]] QoS 等级、运行时不支持）**自动回退到重建**——模式名里的 "OrRecreate" 即此。真要零重建只能用 Off/Initial。

> ⚠️ 别在同一资源维度上**同时**用 HPA 和 VPA（一个想加副本、一个想加单体资源，会打架）。常见组合：HPA 管 CPU 副本伸缩 + VPA 只出内存推荐。

## 勘误 / staleness

- ⚠️ API 现为 **`autoscaling/v2`（GA v1.23）**，schema 用 `metrics[].resource.target.averageUtilization`；源文档的 `autoscaling/v2beta1` 与 `targetAverageUtilization` 已过时。
- ⚠️ **Heapster 已移除**，指标来源是 metrics-server / 自定义 metrics API。

## 相关

- 手动扩缩容与滚动升级：[[concepts/扩缩容与滚动升级]]
- requests/limits 与 QoS（HPA 依赖 requests）：[[concepts/资源限制]]
- 节点级伸缩：Cluster Autoscaler（源文档另章，未单独建页）
