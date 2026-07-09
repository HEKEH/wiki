---
title: Service（资源对象）
date: 2026-06-30
tags: [kubernetes, service, 负载均衡, endpointslices, clusterip]
sources: [kubernetes/objects/service.md]
---

# 源摘要：Service

feisky handbook `/concepts/objects/service.md` 的要点（深入页见 [[entities/Service]]）。

## 负载均衡的四种机制

- **Service**：集群内 L4 负载均衡，借 cloud provider LB 对外。
- **Ingress Controller**：L7 对外（见 [[entities/Ingress]]）。
- **Service Load Balancer**（haproxy 跑容器里，已不推荐）/ **Custom LB**（替代 kube-proxy，接入既有设备）。

## 四种 Service 类型

- **ClusterIP**（默认）：仅集群内可达的虚拟 IP。
- **NodePort**：每台 Node 开端口，`<NodeIP>:NodePort` 可达。
- **LoadBalancer**：在 NodePort 上叠 cloud provider 的外部 LB（物理机可用 MetalLB）。
- **ExternalName**：用 DNS **CNAME** 转发到外部域名，不分配 ClusterIP。

## 关键机制

- **无 selector 的 Service**：手动建 endpoint（或 EndpointSlice）指向集群外 IP，把外部服务纳入 Service。
- **Headless Service（`clusterIP: None`）**：不分配 ClusterIP，DNS 直接返回后端 Pod A 记录列表（StatefulSet 用，见 [[entities/CoreDNS]]）。
- **源 IP 保留**：ClusterIP 内部流量不 SNAT；NodePort/LoadBalancer 默认 SNAT（看到 Node IP），设 `externalTrafficPolicy: Local` 只代理本地 endpoint 以保留真实源 IP（见 [[entities/kube-proxy]]）。
- **`internalTrafficPolicy: Local`**：内部流量只发本节点 endpoint。
- 协议：TCP / UDP / SCTP。

## 较新特性

- ⚠️ **Endpoints API 于 v1.33 弃用**（不移除），新功能（双栈等）只在 **EndpointSlices**（`discovery.k8s.io/v1`，一个 Service 多个 slice，按 `kubernetes.io/service-name` label 查）。
- **多 Service CIDR（v1.33 GA）**：新增 `ServiceCIDR` / `IPAddress` 对象，可动态加多个 ClusterIP 网段。

## 勘误 / staleness

- ⚠️ 示例的 `kube-dns`、`extensions/v1beta1` 已过时——DNS 为 **CoreDNS**，Deployment 用 apps/v1。
- ExternalName "需 kube-dns 1.7+" 已无意义（CoreDNS 早支持）。
