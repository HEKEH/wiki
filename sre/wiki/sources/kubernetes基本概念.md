---
title: Kubernetes 基本概念（源文档摘要）
date: 2026-06-23
tags: [kubernetes, 基本概念, pod, service, label]
sources: [kubernetes/kubernetes基本概念.md]
---

# Kubernetes 基本概念（源文档摘要）

来源：`raw/kubernetes/kubernetes基本概念.md` ｜ 原文：<https://kubernetes.feisky.xyz/introduction/concepts>

介绍 Kubernetes 最核心的几个对象与抽象：

- **Container（容器）**：OS 级虚拟化技术，用 namespace 隔离、用镜像自包含运行环境。详见 [[concepts/容器]]。
- **Pod**：调度的基本单位，一组共享 IPC/Network namespace 的容器集合。详见 [[entities/Pod]]。
- **Node**：Pod 真正运行的主机，需运行 container runtime、kubelet、kube-proxy。详见 [[entities/Node]]。
- **Namespace**：资源对象的抽象集合，用于逻辑隔离（pods/services/deployments 属于 namespace；node/PV 不属于）。详见 [[entities/Namespace]]。
- **Service**：应用服务抽象，按 label 提供负载均衡与服务发现，分配 cluster IP 与 DNS。详见 [[entities/Service]]。
- **Label**：识别对象的 key/value 标签，配合 Label Selector 选择一组对象。详见 [[concepts/Label与Selector]]。
- **Annotations**：key/value 注解，记录附加信息（不用于选择对象）。

所有对象均用 manifest（YAML/JSON）定义，通过 [[entities/kubectl]] 操作。
