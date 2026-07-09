---
title: ResourceQuota / LimitRange（资源对象）
date: 2026-06-30
tags: [kubernetes, resourcequota, limitrange, 配额, 多租户]
sources: [kubernetes/objects/quota.md]
---

# 源摘要：ResourceQuota

feisky handbook `/concepts/objects/quota.md` 的要点（深入页见 [[entities/ResourceQuota]]）。

## 核心

- **ResourceQuota** 限制单个 [[entities/Namespace]] 的资源总用量（每 namespace 最多一个 ResourceQuota）；超额即禁止创建新资源。靠准入控制 `ResourceQuota` 实施。
- 开启计算配额后，创建容器**必须**写 requests/limits（或用 LimitRange 设默认值）。

## 配额类型

- **计算**：`requests.cpu`/`limits.cpu`/`requests.memory`/`limits.memory`。
- **存储**：`requests.storage`、`persistentvolumeclaims`、按 storageclass 限额、`requests/limits.ephemeral-storage`。
- **对象数**：pods、configmaps、secrets、services、`services.loadbalancers`、`services.nodeports`、pvc 等。

## LimitRange（与 ResourceQuota 互补）

- 给 namespace 设容器/Pod 的 **min/max/default/defaultRequest** 资源——ResourceQuota 限总量，LimitRange 限单个对象并补默认值。

## 配额范围（scopes / scopeSelector）

- Terminating / NotTerminating / BestEffort / NotBestEffort；scopeSelector 可按 **PriorityClass** 分层配额。

## 较新

- v1.33 原地 Pod 资源调整会在调整时**校验是否超配额**（原子 + 失败回滚）。

> 注：源文档大量 `count/pods.resize-*` 等字段为示意，非稳定 API，以官方为准。
