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

## PV↔PVC 如何绑定（及重启为何不丢）

**绑定不靠名字硬连,而是 kube-controller-manager 的 PV 控制器做"匹配 + 一次性固化"**:

- **匹配条件**(PV 须同时满足 PVC):`capacity ≥ 请求`、`accessModes` 兼容、`storageClassName` 一致、`volumeMode` 一致、`selector`(若有)命中;多候选选容量最省的。
- **绑定后双向指针、一对一独占**:`PVC.spec.volumeName → PV` 且 `PV.spec.claimRef → PVC`,写进 etcd;PV 变 `Bound` 后不能再绑别的 PVC。
- **三种关联**:动态供给(provisioner 建**专属 PV 预绑定**)、静态匹配(控制器自动挑)、`PVC.spec.volumeName` **点名强绑**。

> **重启为何仍绑回原 PVC、数据不丢**:匹配**只在首次绑定发生一次**,之后 `volumeName`↔`claimRef` 指针**固化在 etcd**、数据固化在**后端盘**(PV 记着 disk ID/NFS path,生命周期独立于集群)。重启后控制器看到"已绑定"直接保持、**不重新匹配**,Pod 挂回同一块盘即恢复。只有**删 PVC** 才按 `reclaimPolicy` 解绑;连 etcd 都丢了还能靠"后端盘 + 建 PV 指向它 + PVC 用 `volumeName` 点名"手动救回。[[entities/StatefulSet]] 的 PVC 名确定性(`data-web-0`)、Pod 重建不删 PVC,故按序号复用回原盘。

## 两个关键字段

**accessModes（访问模式）**——绑定时按"容量 + 访问模式"匹配：

| 模式 | 读/写 | 挂几个**节点** | 同节点多 Pod 共享 | 典型后端 |
| --- | --- | --- | --- | --- |
| **RWO**（ReadWriteOnce） | 读写 | 1 个节点 | **能**（同机多 Pod 可同时读写！） | 云盘 EBS/GCE PD、大多块存储 |
| **ROX**（ReadOnlyMany） | 只读 | 多个节点 | 能 | 只读共享内容 |
| **RWX**（ReadWriteMany） | 读写 | 多个节点 | 能 | **共享文件系统**：NFS/CephFS/Azure Files |
| **RWOP**（ReadWriteOncePod，v1.22+，v1.29 GA） | 读写 | 1 个节点 | **不能**（全集群仅 1 Pod） | 需 CSI |

- **最大的坑**：**RWO 的 "Once" 是"每节点一次"、不是"每 Pod 一次"**——同一节点上多个 Pod **可同时读写**一块 RWO 卷（可能并发写坏），它只保证不跨节点。云盘天然 RWO（一块盘物理上只挂一台 VM）。
- **RWOP 才是"真·单 Pod"**：连同节点的第二个 Pod 也挡掉，用于**必须严格单写者**（如某些数据库）；需 CSI 支持。
- **RWX 必须共享文件系统**：块存储做不到多节点读写，得用 NFS/CephFS/Azure Files。
- 关联 [[entities/Node]] 的 `out-of-service`：RWO 卷卡在宕机节点、[[entities/StatefulSet]] 无法在别处重建，正因"一块盘只能挂一个节点"，必须先强制 detach。

**reclaimPolicy（PVC 释放后怎么处置 PV）**：**Retain**（保留待手动清）/ **Delete**（删后端存储）/ ~~Recycle~~（清空，**已弃用**）。

## StorageClass：动态供给

手动建 PV 在规模大时很烦；StorageClass 让 PVC **按需自动创建 PV**：

- 字段：`provisioner`（存储插件）、`parameters`（插件参数）、`reclaimPolicy`、`mountOptions`、`allowVolumeExpansion`。
- **默认 StorageClass**：经 `DefaultStorageClass` 准入控制 + annotation `storageclass.kubernetes.io/is-default-class=true`，给没写 `storageClassName` 的 PVC 自动套用。
- **`volumeBindingMode`**：决定 PVC **何时建卷/绑定**：
  - **`Immediate`**（默认）：PVC 一创建就**立刻**建卷绑定——**还不知道 Pod 会调度到哪**。
  - **`WaitForFirstConsumer`**：推迟到有 Pod 用它、**正被调度时**才建卷/绑定。调度器（VolumeBinding 插件）**先综合 Pod 的亲和/资源约束选好节点，再在该节点的 zone/节点上建卷** → 卷与 Pod 天然同处。
  - **解决的错配**：云盘不能跨区挂载，`Immediate` 下"卷建在 zone-a、Pod 却被调到 zone-b"会挂不上；本地卷的 PV 钉死某节点，也可能"卷在 node-1、Pod 调到 node-2"。`WaitForFirstConsumer` 让**卷绑定与 Pod 调度一起决策**，是**多可用区**和**本地卷**的关键（本地卷几乎必用）。

## 用起来：挂载示例

**普通 Pod（动态供给,最常见）**——建 PVC（PV 自动生成）+ Pod 挂 PVC，省掉手写 PV：

```yaml
# ① PVC(动态:PV 由 StorageClass 自动建)
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data, namespace: default }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3            # 不写则用默认 SC
  resources: { requests: { storage: 20Gi } }
---
# ② Pod 挂载(volumes 引用 PVC + volumeMounts 挂到路径,靠 name 对接)
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  containers:
  - name: app
    image: postgres
    volumeMounts:
    - { name: data, mountPath: /var/lib/postgresql/data }   # 数据写这里
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: data }              # ← 引用 PVC
```

> 数据默认就扛 Pod 删除/重调度(重新挂回同一块盘);想**删 PVC 也不丢数据**，需 PV 用 `reclaimPolicy: Retain`（默认动态卷是 `Delete`，删 PVC 连盘一起删）。静态供给则先手写 PV，再走同样的 PVC→Pod 挂载（见"用起来"上一节的绑定）。

**有状态应用用 StatefulSet（自动给每个副本建 PVC）**——不必手动建 PVC：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: web }
spec:
  serviceName: web
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx
        volumeMounts:
        - { name: data, mountPath: /usr/share/nginx/html }
  volumeClaimTemplates:            # ← 每个副本自动建一份 PVC:data-web-0/1/2
  - metadata: { name: data }
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources: { requests: { storage: 20Gi } }
```

> `volumeClaimTemplates` 给每个副本生成**确定性命名**的 PVC（`data-web-0`、`data-web-1`…），Pod 重建/重调度按序号**复用回原 PVC→原盘→原数据**（见 [[entities/StatefulSet]]）。这是数据库/有状态服务的标准姿势。

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
