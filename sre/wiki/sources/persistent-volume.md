---
title: PersistentVolume（资源对象）
date: 2026-06-30
tags: [kubernetes, persistentvolume, pvc, storageclass, csi]
sources: [kubernetes/objects/persistent-volume.md]
---

# 源摘要：PersistentVolume

feisky handbook `/concepts/objects/persistent-volume.md` 的要点（深入页见 [[entities/PersistentVolume]]）。

## 核心

- **PV** = 集群级存储资源（独立于 Pod 生命周期）；**PVC** = 对 PV 的申请。Pod 只写 PVC，与具体存储后端解耦。

## 生命周期与状态

- 阶段：Provisioning → Binding → Using → Releasing → Reclaiming/Deleting。
- 状态：**Available / Bound / Released / Failed**。
- `StorageObjectInUseProtection`（v1.11 GA）：删使用中的 PV/PVC 会卡 Terminating 直到使用者释放。

## 关键字段

- **accessModes**：RWO（单节点读写）/ ROX（多节点只读）/ RWX（多节点读写，少数后端如 NFS 支持）。
- **reclaimPolicy**：Retain（保留待手动清）/ Recycle（清空，已弃用）/ Delete（删后端存储）。

## StorageClass（动态供给）

- 免去手动建 PV：`provisioner` + `parameters` + `reclaimPolicy` + `mountOptions`。
- `DefaultStorageClass` 准入控制 + annotation `storageclass.kubernetes.io/is-default-class=true` 设默认。
- **`volumeBindingMode: WaitForFirstConsumer`**：推迟到 Pod 调度后再建/绑卷——本地卷与拓扑感知调度的关键。

## 进阶

- **Local Volume**：本地盘作 PV，靠 `nodeAffinity` 绑定节点，**不支持动态供给**（见 [[entities/PersistentVolume]] Local 一节）。
- **扩容**：StorageClass `allowVolumeExpansion: true` 后可改 PVC 请求扩容。
- **Raw Block Volume**：`volumeMode: Block` 直通块设备。
- **拓扑感知动态供给**（多可用区按 zone 建卷）。
- **Volume Populators（v1.33 GA）**：PVC `dataSourceRef` 用自定义资源预填充数据（备份/模板恢复）。
- **PV 资源泄漏防护（v1.33 GA）**：CSER finalizer 保证乱序删 PV/PVC 时后端存储正确回收。

## 勘误 / staleness

- ⚠️ **Recycle 回收策略已弃用**（用动态供给替代）；in-tree provisioner 大多已迁移 **CSI**；`ExpandPersistentVolumes`/`BlockVolume` 等门控早已 GA。
- 新增 **RWOP（ReadWriteOncePod，v1.22+）** 访问模式源文档未列。
