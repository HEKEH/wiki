---
title: Kubernetes 架构原理（源文档摘要）
date: 2026-06-26
tags: [kubernetes, 架构, borg, 分层架构, 组件]
sources: [kubernetes/kubernetes架构原理.md]
---

# Kubernetes 架构原理（源文档摘要）

来源：`raw/kubernetes/kubernetes架构原理.md` ｜ 原文：<https://kubernetes.feisky.xyz/concepts/architecture>

## 核心要点

- **源自 Borg**：K8s 提供面向应用的容器集群管理，目标是**消除编排计算/网络/存储基础设施的负担**，让开发/运维聚焦于以容器为中心的自助运营；同时作为稳定平台供构建定制工作流。详见 [[entities/Borg]]。
- **整体架构借鉴 Borg**：Pod、Service、Labels、单 Pod 单 IP 等理念均来自 Borg。

## Kubernetes 整体架构

![architecture](../../raw/kubernetes/images/FBSPcbwCX1bdHrjdyApq.png)

核心组件（与 [[entities/Kubernetes]]、[[concepts/控制平面与控制循环]] 一致）：

| 组件 | 职责 |
| --- | --- |
| etcd | 保存整个集群状态 |
| kube-apiserver | 资源操作唯一入口；认证/授权/访问控制/API 注册发现 |
| kube-controller-manager | 维护集群状态：故障检测、自动扩展、滚动更新 |
| kube-scheduler | 按调度策略将 Pod 调度到机器 |
| kubelet | 维持容器生命周期，管理 Volume（CVI）与网络（CNI） |
| Container runtime | 镜像管理及 Pod/容器运行（CRI） |
| kube-proxy | 为 Service 提供集群内服务发现与负载均衡 |

## 推荐 Add-ons

kube-dns（集群 DNS）、Ingress Controller（外网入口）、Heapster（资源监控）、Dashboard（GUI）、Federation（跨集群）、Fluentd-elasticsearch（日志采集存储查询）。

## 分层架构

![分层架构](../../raw/kubernetes/images/0UH3PWeC9VjzAJeyY09a.png)

类似 Linux 的分层：核心层 / 应用层 / 管理层 / 接口层 / 生态系统。详见 [[concepts/分层架构]]。

## 数据陈旧提示

源文档部分内容已过时，引用时注意：

- **"默认容器运行时为 Docker"** —— 已过时；dockershim 于 v1.24 移除，现主流 containerd（见 [[concepts/插件机制与可扩展性]]）。
- **Heapster** 已废弃（由 metrics-server 取代）；**kube-dns** 多被 **CoreDNS** 取代。
- **Federation** v1 已废弃，"跨可用区"说法偏旧（见 [[concepts/集群架构]]）。
- "kube-apiserver 从 v1.33 起流式列表响应优化"为源文档新增的较新细节。
