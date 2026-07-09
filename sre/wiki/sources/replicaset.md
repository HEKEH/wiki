---
title: ReplicaSet / ReplicationController（资源对象）
date: 2026-06-30
tags: [kubernetes, replicaset, replicationcontroller, 副本]
sources: [kubernetes/objects/replicaset.md]
---

# 源摘要：ReplicaSet

feisky handbook `/concepts/objects/replicaset.md` 的要点（深入页见 [[entities/ReplicaSet]]）。

## 核心

- **ReplicationController（RC）**：最早的副本保障器——维持用户定义的副本数（异常退出补、多余的回收）。
- **ReplicaSet（RS）**：RC 的新版，本质相同，区别在 **RS 支持集合式 selector（`matchExpressions`）**，RC 仅支持等式匹配。
- 建议**不直接用 RS**，而交给 [[entities/Deployment]] 托管——获得滚动升级、版本记录、回滚、暂停等高级能力（RS 自身不支持 rolling-update）。

## 勘误 / staleness

- ⚠️ RS 示例用 `apiVersion: extensions/v1beta1`——已过时，正式版为 **apps/v1**。
- ⚠️ RC 现为 **legacy**，新项目一律用 Deployment/ReplicaSet。
- 表头"Deployment 版本"系源文档复制粘贴笔误（实为各对象的 API 版本对照）。
