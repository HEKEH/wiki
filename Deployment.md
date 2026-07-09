---
title: Deployment
date: 2026-06-23
tags: [kubernetes, deployment, replicaset, 滚动升级, 回滚]
sources: [kubernetes/kubernetes 101.md, kubernetes/kubernetes 201.md, kubernetes/objects/deployment.md]
---

# Deployment

Deployment 是管理无状态应用的核心控制器，声明应用的期望状态（副本数、镜像、更新策略），由它经 **ReplicaSet** 自动创建并维护 [[entities/Pod]]。

## 与 ReplicaSet / ReplicationController 的关系

- `kubectl run` 并非直接创建 Pod，而是先创建一个 **Deployment**（replicas=1），再由其关联的 **ReplicaSet** 自动创建 Pod：

  ```bash
  kubectl run --image=nginx:alpine nginx-app --port=80   # 实为创建 Deployment
  ```

- 层级关系：**Deployment → ReplicaSet → Pod**。
- **ReplicationController（RC）** 是更早的副本管理器；`kubectl rolling-update` 仅作用于 RC，而 Deployment 用声明式的 `RollingUpdate` 策略（详见 [[concepts/扩缩容与滚动升级]]）。

## 定义示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    run: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate      # Deployment 默认更新策略
  template:
    metadata:
      labels:
        run: nginx-app
    spec:
      containers:
      - image: nginx
        name: nginx-app
        ports:
        - containerPort: 80
```

## 能力

- **扩缩容**：`kubectl scale --replicas=3 deployment/nginx-app`
- **滚动升级与回滚**：`kubectl set image`、`kubectl rollout status/history/undo`
- **健康检查 / 资源限制**：在 Pod template 的 `spec.containers` 中配置（见 [[concepts/健康检查]]、[[concepts/资源限制]]）。
- **暂停 / 恢复**：`kubectl rollout pause/resume`——暂停期间改模板**不触发 rollout**，可攒多次修改一次发布；暂停态不可回滚。

滚动升级触发条件与命令详见 [[concepts/扩缩容与滚动升级]]。

## 更新策略 `.spec.strategy`

| 类型 | 行为 | 取舍 |
| --- | --- | --- |
| **RollingUpdate**（默认） | 新建+逐步替换，`maxSurge`（最多超额）/`maxUnavailable`（最多不可用）控节奏（现默认各 **25%**） | 无中断，但升级期间新旧版本并存 |
| **Recreate** | 先杀光旧 Pod 再建新的 | 有停机，但保证同一时刻只有一个版本 |

## 进阶行为

- **比例扩容（proportional scaling）**：在一次 rollout *进行中*又扩容时，新增副本按比例分摊到新旧 active ReplicaSet（而非全堆新 RS），降低风险。
- **状态 conditions**：`Progressing` / `Available` / `ReplicaFailure`。`progressDeadlineSeconds` 超时未推进 → `Reason=ProgressDeadlineExceeded`（K8s 仅报告、不自动回滚）。
- **清理**：`revisionHistoryLimit` 限制保留的旧 RS 数（现默认 **10**）；设 0 则**无法回滚**。

> ⚠️ 源文档示例的 `--record`、`extensions/v1beta1` 已过时（`--record` 废弃，正式版 **apps/v1**）；"未来 maxSurge/maxUnavailable 改 25%、revisionHistoryLimit 默认 2"等表述已成现实（现为 25%/25%、保留 10）。
