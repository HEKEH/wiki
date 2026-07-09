---
title: LocalVolume（资源对象）
date: 2026-06-30
tags: [kubernetes, local-volume, 本地存储, pv, nodeaffinity]
sources: [kubernetes/objects/local-volume.md]
---

# 源摘要：LocalVolume

feisky handbook `/concepts/objects/local-volume.md` 的要点（并入深入页 [[entities/PersistentVolume]] Local 一节）。

## 核心

- 本地数据卷代表节点上的**本地磁盘/分区/目录**，用于分布式存储、数据库等高性能高可靠场景；支持块设备与文件系统（`spec.local.path`）。
- 只能以**静态 PV** 使用（不支持动态供给），靠 **`nodeAffinity`** 绑定到指定节点。
- 对比 hostPath：local 是**真正持久化**、并保证 Pod 总调度回该节点；hostPath 仅简单挂载、无此保证。
- StorageClass 用 `provisioner: kubernetes.io/no-provisioner` + `volumeBindingMode: WaitForFirstConsumer`；社区 `local-volume-provisioner` 可自动建/清。

## 最佳实践

- 每卷独立磁盘（IO 隔离）/独立分区（容量隔离）；用 UUID 或 `/dev/disk/by-id` 而非裸路径；避免重建同名 Node（否则新 Node 认不回旧 PV）。

## 勘误 / staleness

- ⚠️ "v1.10 Beta" 已过时——Local Volume 自 **v1.14 GA**；文中"计划 v1.9 支持"等 TODO 均已落地。`kubernetes-incubator/external-storage` 已迁至 sig-storage。
