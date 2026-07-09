---
title: Volume（资源对象）
date: 2026-06-30
tags: [kubernetes, volume, emptydir, hostpath, projected, csi]
sources: [kubernetes/objects/volume.md]
---

# 源摘要：Volume

feisky handbook `/concepts/objects/volume.md` 的要点（深入页见 [[concepts/Volume存储]]）。

## 核心

- Volume 生命周期与 [[entities/Pod]] 绑定：**容器重启数据还在；Pod 删除才清理**，是否丢数据取决于类型（emptyDir 丢、PV 不丢）。

## 常见类型

- **emptyDir**：Pod 在某 Node 上的临时空目录，Pod 删除/迁移即永久丢失（容器挂掉不丢）。
- **hostPath**：挂 Node 上的文件/目录（Node 级，跟节点走）。
- **配置注入**：`secret` / `configMap` / `downwardAPI`。
- **网络/云**：nfs、cephfs、rbd、glusterfs、iscsi、gce/aws/azure 盘等。
- **`persistentVolumeClaim`**：解耦的持久卷申请（见 [[entities/PersistentVolume]]）。
- **`image`**（v1.33 Beta）：把容器镜像作为只读卷挂入（`ImageVolume` 门控）。

## 重要机制

- **subPath**：把卷里某个文件/子目录挂到目标路径，**不覆盖**整个目录。
- **Projected Volume**：把 secret/configMap/downwardAPI 多源映射到同一目录。
- **MountPropagation**（v1.10 Beta）：None（私有）/ HostToContainer（rslave）/ Bidirectional（rshared，仅特权容器）。

## 勘误 / staleness

- ⚠️ **in-tree 云存储插件**（gcePersistentDisk/awsElasticBlockStore/azureDisk/vsphereVolume 等）已弃用并迁移到 **CSI**（多数 in-tree 实现已移除）；**gitRepo 已弃用移除**；**FlexVolume 已弃用**（用 CSI）。
- ⚠️ 本地存储限额示例的 `storage.kubernetes.io/overlay`/`scratch` 旧模型已被统一资源 **`ephemeral-storage`** 取代（见 [[entities/kubelet]] 驱逐）。
