---
title: Node
date: 2026-06-23
tags: [kubernetes, node, 服务节点]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes集群.md]
---

# Node

Node 是 [[entities/Pod]] 真正运行的主机，可以是物理机也可以是虚拟机。在集群中扮演**服务节点**角色——真正运行容器、管理镜像，并提供集群内的服务发现与负载均衡。

## 必备组件

为了管理 Pod，每个 Node 至少要运行：

- **container runtime**（如 docker、rkt，现主流为 containerd）—— 实际运行容器。
- **[[entities/kubelet]]** —— 维护容器生命周期，管理 Volume（CVI）与网络（CNI）。
- **kube-proxy** —— 为 [[entities/Service]] 提供集群内的服务发现与负载均衡。

## 在集群中的位置

一个 Kubernetes 集群由**控制节点（controller）**、**服务节点（Node）**和**分布式存储 etcd** 组成。Node 即服务节点。详见 [[concepts/集群架构]]。

## 备注

部分资源**不属于任何 namespace**，Node 即是其中之一（另有 persistentVolumes 等）。见 [[entities/Namespace]]。
