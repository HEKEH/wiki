---
title: Kubernetes 101（源文档摘要）
date: 2026-06-23
tags: [kubernetes, kubectl, 入门, pod, volume, service]
sources: [kubernetes/kubernetes 101.md]
---

# Kubernetes 101（源文档摘要）

来源：`raw/kubernetes/kubernetes 101.md` ｜ 原文：<https://kubernetes.feisky.xyz/introduction/101>

入门级实操，以运行一个 nginx 为例：

- **kubectl 基本操作**：`kubectl run`（类比 `docker run`）创建容器——实际创建的是由 [[entities/Deployment]] 管理的 Pod；`get`/`describe`/`logs`/`exec` 分别类比 docker 的 `ps`/`inspect`/`logs`/`exec`。详见 [[entities/kubectl]]。
- **用 YAML 定义 Pod**：`kubectl create -f file.yaml` 创建资源；`kubectl run` 实为创建 Deployment(replicas=1) → ReplicaSet → Pod。
- **使用 Volume**：Pod 短暂、数据随之消失，用 Volume 持久化；K8s 支持非常多卷插件（emptyDir/hostPath/nfs/cephfs/PVC 等）。详见 [[concepts/Volume存储]]。
- **使用 Service**：Pod IP 会变，用 Service 提供统一入口、负载均衡与服务发现；`kubectl expose ... --type=NodePort` 暴露服务，ClusterIP 内部访问、NodePort 内外均可访问。详见 [[entities/Service]]。
