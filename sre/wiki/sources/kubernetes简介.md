---
title: Kubernetes 简介（源文档摘要）
date: 2026-06-23
tags: [kubernetes, 容器编排, 概述]
sources: [kubernetes/kubernetes简介.md]
---

# Kubernetes 简介（源文档摘要）

来源：`raw/kubernetes/kubernetes简介.md`

## 核心要点

- **来历**：Kubernetes 是谷歌开源的容器集群管理系统，源自 Google 内部大规模容器管理技术 **Borg**，现已成为容器编排领域的领导者。
- **主要功能**：容器化应用的部署/维护/滚动升级、负载均衡与服务发现、跨机器跨地区集群调度、自动伸缩、有/无状态服务、广泛的 Volume 支持、插件扩展机制。
- **定位为平台**：通过 Label/Annotation 组织资源；控制器构建在与用户相同的 API 之上，用户可编写自定义控制器/调度器并通过插件扩展。详见 [[concepts/容器编排]]。
- **不是什么**：不是包罗万象的 PaaS；不限制语言/框架（应用符合 [12 因素](http://12factor.net/) 即可）；不内置中间件/数据库/存储；不提供服务市场、配置语言或机器维护自愈系统。这些都直接运行在 K8s 之上（Openshift、Deis 等 PaaS 即构建于其上）。
- **声明式本质**：通过声明式 API + 一组独立、可组合的控制器，保证应用始终处于期望状态，用户无需关心中间状态如何转换 —— 这正是其可靠性与弹性的根基。详见 [[concepts/声明式API]]。

## 核心组件

| 组件 | 职责 |
| --- | --- |
| etcd | 保存整个集群的状态 |
| apiserver | 资源操作唯一入口；认证、授权、访问控制、API 注册与发现 |
| controller manager | 维护集群状态：故障检测、自动扩展、滚动更新 |
| scheduler | 按调度策略将 Pod 调度到相应机器 |
| kubelet | 维护容器生命周期，管理 Volume（CVI）与网络（CNI） |
| Container runtime | 镜像管理及 Pod/容器的实际运行（CRI） |
| kube-proxy | 为 Service 提供集群内服务发现与负载均衡 |

详见实体页 [[entities/Kubernetes]]。

## 数据陈旧提示

源文档的版本支持表仅覆盖 v1.6.x（2017-03）至 v1.11.x，且声称稳定版发布后支持 9 个月 —— 这些信息已严重过期（当前 K8s 支持周期已延长至约 14 个月）。引用版本/支持周期时应以官方文档为准。

## 参考

- [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [HOW CUSTOMERS ARE REALLY USING KUBERNETES](https://apprenda.com/blog/customers-really-using-kubernetes/)
