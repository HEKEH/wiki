---
title: Ingress（资源对象）
date: 2026-06-30
tags: [kubernetes, ingress, l7, 路由, tls, ingressclass]
sources: [kubernetes/objects/ingress.md]
---

# 源摘要：Ingress

feisky handbook `/concepts/objects/ingress.md` 的要点（深入页见 [[entities/Ingress]]）。

## 核心

- Ingress 是**进入集群请求的 L7 路由规则集合**：给 Service 提供对外 URL、按 host/path 路由、SSL 终止、负载均衡。
- Ingress 本身**只是规则**，必须有 **Ingress Controller**（以 Pod/DaemonSet 跑、watch Ingress 与 Service）才真正配置 LB。Controller 需用户自选/自部署（nginx、GCE、Traefik 等），**不随 controller-manager 自启**。

## 路由类型

- **单服务**（默认后端，可做 404 页）/ **多路径**（`/foo`→s1、`/bar`→s2）/ **虚拟主机**（按 host 头分流，共用 IP）/ **TLS**（从 Secret 取 `tls.crt`/`tls.key` 做 TLS 终止，多 host 靠 SNI 复用）。

## IngressClass

- 旧法用 annotation `kubernetes.io/ingress.class: nginx` 选 controller；新法用 **IngressClass** 对象（`spec.controller`）预声明、Ingress 直接引用。

## 演进

- **Gateway API** 是 Ingress 的下一代（见 [[entities/GatewayAPI]]），协议更广、角色分离、原生扩展；建议新项目采用。

## 勘误 / staleness

- ⚠️ 全文示例用 `apiVersion: extensions/v1beta1` 与 `serviceName/servicePort` 旧字段——已过时：Ingress **v1.19 起为 `networking.k8s.io/v1`（GA）**，后端字段改为 `backend.service.name` + `service.port.number`，路径需 `pathType`。
