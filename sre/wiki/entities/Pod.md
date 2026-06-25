---
title: Pod
date: 2026-06-23
tags: [kubernetes, pod, 容器]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes 101.md]
---

# Pod

Pod 是 Kubernetes **调度的基本单位**，是一组紧密关联的[[concepts/容器]]集合，每个 Pod 可包含一个或多个容器。

## 为什么需要 Pod 这一层

Pod 是介于"单个容器"与"整台 [[entities/Node]]"之间、有意设计的**最小调度与共享单位**。即使大多数 Pod 只含一个容器，这层抽象仍解决了直接在 Node 上跑容器无法解决的问题：

- **表达"一组容器是一伙的"**：紧密协作的辅助容器（sidecar，如日志收集、服务网格代理、配置同步）必须与主容器同机、共享网络和文件才能工作。只有"单容器"这一个单位时，无法声明这种归属关系。
- **原子调度**：Pod 内的容器保证被**一起调度到同一 Node、同生共死**，不会被分到不同机器而导致 localhost / 共享卷失效。
- **简化网络模型**：K8s 采用"**每个 Pod 一个 IP**"（而非每容器一个），Pod 内容器共享该 IP，避免端口冲突与 IP 数量爆炸；[[entities/Service]] 面向的也是 Pod IP。
- **统一挂载点**：[[concepts/Volume存储]]、生命周期、[[concepts/健康检查]] 都以 Pod 为单位统一管理，Pod 内多容器可共享同一卷。
- **与运行时解耦**：Pod 是运行时无关的抽象，K8s 只管调度/管理 Pod，容器具体用哪个运行时跑交给 CRI 插件（见 [[concepts/插件机制与可扩展性]]），换运行时不影响核心模型。

> 一句话：直接在 Node 上跑容器，缺的是"一组容器共享网络/存储/命运、并作为整体被调度"的能力，Pod 正是为此而生。

## 关键特性

- **共享命名空间**：Pod 内的多个容器共享 IPC 和 Network namespace，因此可通过进程间通信（IPC）和文件共享高效协作。
- **共享网络与文件系统**：同一 Pod 内的容器共享网络（同一 IP/端口空间）和挂载的 [[concepts/Volume存储]]。每个 Pod 有独立 IP（含同机 Pod），详见 [[concepts/Pod网络模型]]。
- **调度单位**：K8s 调度的是 Pod 整体，而非单个容器——同一 Pod 的容器总在同一 [[entities/Node]] 上运行。
- **生命周期短暂**：Pod 出现异常时会被销毁并由新 Pod 替代；其 IP 会随重启变化，因此不应直接用 Pod IP 交互（用 [[entities/Service]] 代替）。

## 定义方式

所有 K8s 对象都用 manifest（YAML 或 JSON）定义。一个最简 nginx Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

通过 `kubectl create -f nginx.yaml` 创建。

## 相关

- 通常不直接创建 Pod，而是由 [[entities/Deployment]] → ReplicaSet 来管理。
- 用 [[concepts/Label与Selector]] 被 Service / 控制器选中。
- 操作工具见 [[entities/kubectl]]。
