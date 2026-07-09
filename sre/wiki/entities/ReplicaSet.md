---
title: ReplicaSet
date: 2026-06-30
tags: [kubernetes, replicaset, replicationcontroller, 副本控制器]
sources: [kubernetes/objects/replicaset.md, kubernetes/设计理念.md]
---

# ReplicaSet

ReplicaSet（rs）是**保证指定数量 [[entities/Pod]] 副本始终在线**的控制器：少了补、多了回收。它是 [[entities/Deployment]] 的底层实现，几乎不单独使用。

## 与 ReplicationController 的关系

| | ReplicationController（rc） | ReplicaSet（rs） |
| --- | --- | --- |
| 定位 | 最早的副本保障器 | rc 的新版，本质相同 |
| selector | 仅**等式**（`app=nginx`） | 支持**集合式**（`matchExpressions`，如 `in (a,b)`，见 [[concepts/Label与Selector]]） |
| 现状 | **legacy**，新项目勿用 | 当前标准，但仍建议交给 Deployment 托管 |

## 为何不直接用 RS

RS 自身**不支持 rolling-update**，也无版本记录/回滚/暂停。这些"复合操作"由 [[entities/Deployment]] 提供——Deployment 通过新建/伸缩多个 RS 完成滚动升级（见 [[concepts/扩缩容与滚动升级]]）。链路：**Deployment → ReplicaSet → Pod**。

> ⚠️ 源文档示例用 `apiVersion: extensions/v1beta1`，已过时——正式版为 **apps/v1**。

## 相关

- 控制器全景与选型：[[concepts/工作负载控制器]]
- 上层管理者：[[entities/Deployment]]
