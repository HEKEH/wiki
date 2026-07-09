---
title: PersistentVolume (PV/PVC)
date: 2026-06-30
tags: [kubernetes, persistentvolume, pvc, storageclass, csi, 本地卷]
sources: [kubernetes/objects/persistent-volume.md, kubernetes/objects/local-volume.md]
---

# PersistentVolume (PV / PVC / StorageClass)

PV 与 PVC 是把"用存储的人"和"存储是什么技术"**解耦**的一对抽象（概念动机与"计算侧对称"见 [[concepts/Volume存储]]）：

- **PersistentVolume（PV）**：集群级存储资源，生命周期**独立于 [[entities/Pod]]**，由管理员或动态供给创建。
- **PersistentVolumeClaim（PVC）**：使用者的存储申请（要多大、什么访问模式），Pod 只引用 PVC。

## 生命周期与状态

阶段：**Provisioning**（建 PV，静态或动态）→ **Binding**（PV↔PVC 绑定）→ **Using** → **Releasing** → **Reclaiming/Deleting**。

PV 状态：**Available**（待绑）/ **Bound**（已绑）/ **Released**（PVC 删了但还没回收）/ **Failed**。

> `StorageObjectInUseProtection`（v1.11 GA）：删使用中的 PV/PVC 不会立即删，而是卡在 `Terminating` 直到使用者释放——防止"把正在用的盘删了"。

## 两个关键字段

**accessModes（访问模式）**——绑定时按"容量 + 访问模式"匹配：

| 模式 | 含义 |
| --- | --- |
| **RWO**（ReadWriteOnce） | 单**节点**读写（最常见） |
| **ROX**（ReadOnlyMany） | 多节点只读 |
| **RWX**（ReadWriteMany） | 多节点读写（少数后端如 NFS 支持） |
| **RWOP**（ReadWriteOncePod，v1.22+） | 单 **Pod** 读写（比 RWO 更严，源文档未列） |

**reclaimPolicy（PVC 释放后怎么处置 PV）**：**Retain**（保留待手动清）/ **Delete**（删后端存储）/ ~~Recycle~~（清空，**已弃用**）。

## StorageClass：动态供给

手动建 PV 在规模大时很烦；StorageClass 让 PVC **按需自动创建 PV**：

- 字段：`provisioner`（存储插件）、`parameters`（插件参数）、`reclaimPolicy`、`mountOptions`、`allowVolumeExpansion`。
- **默认 StorageClass**：经 `DefaultStorageClass` 准入控制 + annotation `storageclass.kubernetes.io/is-default-class=true`，给没写 `storageClassName` 的 PVC 自动套用。
- **`volumeBindingMode: WaitForFirstConsumer`**：推迟到 **Pod 被调度后**再建/绑卷——这样卷能建在 Pod 落点所在的 zone/节点，是**本地卷**和**拓扑感知调度**的关键（否则可能卷在 A 区、Pod 被调到 B 区）。

## Local Volume（本地持久卷）

把节点本地磁盘/分区/目录作 PV，用于数据库等要高 IO 的场景：

- **只能静态创建**（不支持动态供给），靠 **`nodeAffinity`** 把 PV 钉在某节点；配 `WaitForFirstConsumer` 让 Pod 调度跟着卷走。
- 对比 hostPath：local 是**真正持久化 + 保证 Pod 总回到该节点**；hostPath 只是简单挂载、无调度保证（见 [[concepts/Volume存储]]）。
- 实践：每卷独立盘/分区做隔离；用 UUID 或 `/dev/disk/by-id` 而非裸路径；别重建同名 Node。

## 进阶能力

- **在线扩容**：StorageClass 设 `allowVolumeExpansion: true` 后，改大 PVC 请求即扩容（不丢数据、不重启）。
- **Raw Block Volume**：`volumeMode: Block` 直通块设备（数据库裸盘）。
- **拓扑感知动态供给**：多可用区按 zone 建卷并与 Pod 反亲和配合。
- **Volume Snapshot**：经 CSI 给卷打快照（`VolumeSnapshot`/`VolumeSnapshotClass`，`snapshot.storage.k8s.io`），可从快照恢复出新 PVC（`dataSource` 指向 VolumeSnapshot）。
- **Volume Populators（v1.33 GA）**：PVC `dataSourceRef` 用自定义资源**预填充**数据（从备份/模板恢复）。
- **PV 资源泄漏防护（v1.33 GA）**：CSI finalizer 保证乱序删 PV/PVC 时后端存储被正确回收。

> ⚠️ 勘误：**Recycle 已弃用**；**in-tree 存储插件大多迁移到 CSI**（见 [[concepts/插件机制与可扩展性]]）；`ExpandPersistentVolumes`/`BlockVolume` 等门控早已 GA；StorageClass 正式版 `storage.k8s.io/v1`。

## 相关

- 卷的总览与"为什么需要"：[[concepts/Volume存储]]
- 谁来挂卷/做存储接口：[[entities/kubelet]]（Volume Manager、CSI）
- 有状态应用经 volumeClaimTemplates 各绑一份 PVC：[[entities/StatefulSet]]
