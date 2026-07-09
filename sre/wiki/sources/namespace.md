---
title: Namespace（资源对象）
date: 2026-06-30
tags: [kubernetes, namespace, 隔离, kube-system]
sources: [kubernetes/objects/namespace.md]
---

# 源摘要：Namespace

feisky handbook `/concepts/objects/namespace.md` 的要点（深入页见 [[entities/Namespace]]）。

## 核心

- 一组资源/对象的抽象集合，用于按项目组/用户组划分。pod/service/deployment 等属某 namespace（默认 `default`）；**node、persistentVolume、namespace 自身不属于任何 namespace**。
- K8s 自带服务跑在 `kube-system`；v1.7 增 `kube-public`（放公共信息如 `cluster-info` ConfigMap）。

## 操作

- `kubectl -n <ns>` / `--all-namespaces`；状态 **Active / Terminating**。
- 名称满足 `[a-z0-9]([-a-z0-9]*[a-z0-9])?`、≤63 字符。
- **删除 namespace 会级联删除其下所有资源**；`default`/`kube-system` 不可删。
- PV 不属 namespace，但 PVC 属；Event 是否属取决于其对象。

> 隔离边界（管理隔离 vs 网络不隔离）、跨 namespace DNS 访问、配额见 [[entities/Namespace]]。
