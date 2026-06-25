---
title: Kubernetes 集群（源文档摘要）
date: 2026-06-23
tags: [kubernetes, 集群, 架构, etcd, minikube, federation]
sources: [kubernetes/kubernetes集群.md]
---

# Kubernetes 集群（源文档摘要）

来源：`raw/kubernetes/kubernetes集群.md` ｜ 原文：<https://kubernetes.feisky.xyz/introduction/cluster>

- **集群组成**：分布式存储 etcd + 控制节点（controller，负责调度/维护状态/扩展/滚动更新）+ 服务节点（[[entities/Node]]，运行容器、管理镜像、服务发现与负载均衡）。详见 [[concepts/集群架构]]。
- **集群联邦（Federation）**：用于跨可用区的多集群，需配合云服务商（GCE、AWS）实现。
- **创建集群**：生产用部署指南；学习/验证可用 **minikube**（单机版，`minikube start`）或 **Play with Kubernetes**（在线、kubeadm、每次最长 4 小时、自动显示 NodePort 端口）。
