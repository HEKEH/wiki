---
title: Volume 存储
date: 2026-06-23
tags: [kubernetes, volume, 存储, 持久化, csi]
sources: [kubernetes/kubernetes 101.md, kubernetes/kubernetes简介.md, kubernetes/objects/volume.md, kubernetes/objects/persistent-volume.md]
---

# Volume 存储

容器内的数据会随 [[entities/Pod]] 消亡而消失（Pod 生命周期短暂，异常即被新 Pod 替代）。**Volume（卷）就是为持久化容器数据、以及在 Pod 内多容器间共享文件而生**。

## 为什么需要

- **持久化**：让数据生命周期独立于容器，Pod 重建后数据仍在。
- **共享**：同一 Pod 内多个容器可挂载同一 Volume 互传文件。

## 使用方式

在 Pod 的 `spec` 中定义 `volumes`，并在容器中通过 `volumeMounts` 挂载。例：为 redis 指定 hostPath 存储数据：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-persistent-storage
      mountPath: /data/redis
  volumes:
  - name: redis-persistent-storage
    hostPath:
      path: /data/
```

## "广泛的 Volume 支持"

K8s Volume 是一套**插件化的存储抽象**（见 [[concepts/插件机制与可扩展性]]），支持非常多后端，按用途分：

- **临时/节点本地**：`emptyDir`（Pod 删即丢）、`hostPath`（挂 Node 路径，**对调度器透明**——Pod 漂到别的节点会挂到那台机的同名路径、数据可能空/错，故只宜读节点系统文件或单机测试）、`local`（**本地持久卷**：登记为 PV + `nodeAffinity` 钉住节点，调度器保证 Pod 回到有数据的节点，见 [[entities/PersistentVolume]]）
- **配置注入**：`secret`（[[entities/Secret]]）、`configMap`（[[entities/ConfigMap]]）、`downwardAPI`（把元数据当文件挂入）
- **网络/分布式存储**：`nfs`、`iscsi`、`glusterfs`、`cephfs`、`rbd`
- **持久卷抽象**：`persistentVolumeClaim`(PVC) —— 应用只声明"要多少存储"，由 PVC/PV 与具体后端解耦
- **镜像卷**：`image`（v1.33 Beta，把容器镜像作只读卷挂入）

> 工程基础是 **CSI（Container Storage Interface）**：厂商实现 CSI 即可接入，对应 [[entities/kubelet]] 职责里的卷管理。这正是 K8s "构建生态平台"定位在存储上的体现。in-tree 云存储插件（gce/aws/azure 盘等）已弃用、迁移到 CSI。

### 临时 vs 持久（最该先分清的一刀）

| | 生命周期 | 例子 |
| --- | --- | --- |
| **临时**（随 Pod 消亡） | 与 Pod 同生死 | `emptyDir`、`configMap`、`secret`、`downwardAPI` |
| **持久**（独立于 Pod） | 独立，重建仍在 | `persistentVolumeClaim`(PV)、`local`、`nfs`、云盘 |

> 两个常用挂载技巧：**`subPath`** 把卷里单个文件/子目录挂到目标路径而**不覆盖**原目录；**Projected Volume** 把 secret/configMap/downwardAPI 多源合并到同一目录。

**注意**：普通 Volume 的生命周期与作用范围是**一个 [[entities/Pod]]**（Pod 内所有容器共享）；而下面的 PV 生命周期**独立于 Pod**。

## PV 与 PVC：存储的逻辑抽象

`persistentVolumeClaim`（PVC）背后是一对解耦的抽象，让"用存储的人"无需关心"存储具体是什么技术"：

- **PersistentVolume（PV）**：实际存储资源，由**集群管理员**配置（对接 AWS/GCE/Ceph/NFS 等具体后端）。
- **PersistentVolumeClaim（PVC）**：使用者的**存储申请**（"我要 10Gi、可读写"），由**服务的管理员/使用者**声明。

二者关系与"计算"侧高度对称：

| 存储侧 | 计算侧 | 角色 |
| --- | --- | --- |
| **PV** | **[[entities/Node]]** | 资源**提供者**，随基础设施变化，由集群管理员配置 |
| **PVC** | **[[entities/Pod]]** | 资源**使用者**，随业务需求变化，由服务管理员配置 |

> 这样应用的配置里只写 PVC（要多少），具体后端技术的配置交给管理员通过 PV 完成——存储的"声明式 + 关注点分离"，呼应 [[concepts/声明式API]]。

PV/PVC 的完整机制（生命周期与状态、accessModes、reclaimPolicy、**StorageClass 动态供给**、`WaitForFirstConsumer`、本地卷、扩容、Raw Block、Volume Populators 等）见 [[entities/PersistentVolume]]。
