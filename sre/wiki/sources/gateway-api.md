---
title: Gateway API（资源对象）
date: 2026-06-30
tags: [kubernetes, gateway-api, l7, 路由, ingress演进]
sources: [kubernetes/objects/gateway-api.md, kubernetes/objects/ingress.md]
---

# 源摘要：Gateway API

feisky handbook `/concepts/objects/gateway-api.md` 的要点（深入页见 [[entities/GatewayAPI]]）。

## 核心

- SIG-NETWORK 维护的 **Ingress 下一代**：表达力强、可扩展、**面向角色**。解决 Ingress 的三大局限：只能简单 HTTP 路由、靠注解扩展、无角色分离。

## 三个核心资源（+ 角色分离）

- **GatewayClass**（类比 StorageClass，由**基础设施提供者**管）：声明一类网关 + 控制器。
- **Gateway**（**集群操作员**管）：定义 listeners（端口/协议/hostname）。
- **HTTPRoute**（**应用开发者**管）：`parentRefs` 挂到 Gateway，按 host/path 路由到 `backendRefs`。
- 路由类型还有 TLSRoute / TCPRoute / UDPRoute / GRPCRoute。

## 与 Ingress 对比

| | Ingress | Gateway API |
| --- | --- | --- |
| 协议 | 仅 HTTP/HTTPS | HTTP/TCP/UDP/TLS/gRPC |
| 角色分离 | 无 | 三层清晰分离 |
| 扩展 | 靠注解 | 原生 API |

## v1.3.0（2025-04）新特性

- 标准通道：**按百分比请求镜像**（蓝绿/压测）。
- 实验通道：**CORS 过滤**、**XListenerSets**（跨 ns 监听器合并）、**重试预算 XBackendTrafficPolicy**、**Inference Extension**（InferencePool/InferenceModel，为 LLM 推理做模型感知路由 + GPU 优化）。

## 兼容性

- 要求 **K8s 1.26+**；标准通道已达 **v1 稳定**；实现有 Envoy Gateway、Istio、Cilium、Airlock 等。
