---
title: Kubernetes
date: 2026-06-23
tags: [kubernetes, 容器编排, 集群管理]
sources: [kubernetes/kubernetes简介.md]
---

# Kubernetes

Kubernetes（K8s）是谷歌开源的**容器集群管理系统**，是 Google 内部大规模容器管理技术 **Borg** 的开源版本，现为容器编排领域的事实标准。

## 主要能力

- 基于容器的应用部署、维护与滚动升级
- 负载均衡与服务发现
- 跨机器、跨地区的集群调度
- 自动伸缩
- 有状态与无状态服务
- 广泛的 Volume 支持
- 插件机制保证可扩展性

## 设计哲学

K8s 是一个**构建生态系统的平台**，而非传统包罗万象的 PaaS：

- 不限制应用类型、语言或框架，只要应用符合 [12 因素](http://12factor.net/) 且能在容器中运行。
- 不内置中间件、数据处理框架、数据库或集群存储 —— 这些应用直接运行在 K8s 之上。
- 控制器构建在与开发者相同的 API 之上，用户可编写自定义控制器/调度器并通过插件扩展。
- 通过 [[concepts/声明式API]] 消除"编排"的需要：用户声明期望状态，系统自动收敛，无需关心中间状态转换。

### K8s 明确"不做"的事（边界）

- 不提供点击即部署的服务市场。
- 不直接构建或部署代码，但可在其上搭建 CI 工作流。
- 允许用户自选日志、监控、告警系统，不强制内置。
- 不提供应用配置语言/系统（如 jsonnet）。
- **不提供机器配置、维护、管理或自愈系统** —— 注意这里的"自愈"指**机器/基础设施级**。K8s 仍提供**应用/Pod 级自愈**（控制器检测到 Pod 故障会自动重建/重调度），二者层级不同，并不矛盾。详见 [[concepts/声明式API]]。

参见概念页 [[concepts/容器编排]]。

## 核心组件

- **etcd** —— 保存整个集群的状态（唯一数据源）。
- **apiserver** —— 资源操作的唯一入口，提供认证、授权、访问控制、API 注册与发现。
- **controller manager** —— 维护集群状态：故障检测、自动扩展、滚动更新。
- **scheduler** —— 按调度策略将 Pod 调度到相应机器。
- **kubelet** —— 维护容器生命周期，管理 Volume（CVI）与网络（CNI）。
- **Container runtime** —— 镜像管理及 Pod/容器的实际运行（CRI）。
- **kube-proxy** —— 为 Service 提供集群内的服务发现与负载均衡。

> 组件可粗分为**控制平面**（etcd、apiserver、controller manager、scheduler）与**节点组件**（kubelet、container runtime、kube-proxy）。

## 相关

- 源文档摘要：[[sources/kubernetes简介]]
