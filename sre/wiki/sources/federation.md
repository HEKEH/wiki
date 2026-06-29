---
title: 源文档 - Federation（集群联邦）
date: 2026-06-27
tags: [kubernetes, federation, 集群联邦, source, 已废弃]
sources: [kubernetes/components/federation.md]
---

# 源文档摘要：Federation（集群联邦）

详述集群联邦（Federation v1）的部署与使用——跨 Region/服务商管理多个 K8s 集群。

## 关键内容

- **定位**：K8s 单集群限于同一地域；Federation 为跨 Region/跨云提供统一控制面。
- **组件**：federation-apiserver、federation-controller-manager、kubefed（CLI）。
- **机制**：经 federation apiserver 注册成员集群（`kubefed join`），创建联邦对象时在所有子集群各建一份；集群间用 DNS 负载均衡。
- **部署**：`kubefed init`（DNS provider、物理机 NodePort、自定义 etcd PV）、ClusterSelector annotation、策略调度（OPA）。
- **联邦资源**：Federated ConfigMap/Service/Deployment/Ingress/Namespace/Secret/Job/HPA 等。

## 勘误 / 时效性

- ⚠️ **重度过时**：Federation **v1（kubefed）已废弃**，后继 KubeFed v2 也已归档。当前多集群管理生态以 **Karmada**、Cluster API、GitOps 等为主。
- ⚠️ 文中"K8s 设计定位单一集群在同一地域"是早期框架；如今跨 AZ 部署是常规（见 [[concepts/集群架构]] 中"跨可用区"勘误）。
- 本页仅作历史/概念参考，**不应作为现行多集群方案**。

## 关联

[[concepts/集群架构]]（集群联邦与跨可用区说明） · [[entities/Kubernetes]]
