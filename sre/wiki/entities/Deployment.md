---
title: Deployment
date: 2026-06-23
tags: [kubernetes, deployment, replicaset, 滚动升级]
sources: [kubernetes/kubernetes 101.md, kubernetes/kubernetes 201.md]
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

详见 [[concepts/扩缩容与滚动升级]]。
