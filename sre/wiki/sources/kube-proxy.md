---
title: 源文档 - kube-proxy
date: 2026-06-27
tags: [kubernetes, kube-proxy, iptables, ipvs, nftables, source]
sources: [kubernetes/components/kube-proxy.md]
---

# 源文档摘要：kube-proxy

详述 Service 数据面的四种代理模式与 iptables/ipvs 实现细节。回答了 home.md "Service iptables/IPVS 转发"开放问题。

## 关键内容

- **四种模式**：userspace（最早，慢）、iptables（长期默认，规则膨胀）、ipvs（v1.11 GA，增量、连接不断）、nftables（O(1) 路由，需内核 5.13+）；winuserspace 限 Windows。
- **iptables 链路**：`KUBE-SERVICES`→`KUBE-SVC-xxx`（按概率随机分流）→`KUBE-SEP-yyy`（DNAT 到 Pod IP）；NodePort/LB/MASQUERADE/`externalTrafficPolicy: Local` 处理，附完整 iptables 规则示例。
- **ipvs**：内核 IPVS 转发（`ipvsadm -ln`），仍用 iptables+ipset 做 SNAT 与规则归类。
- **局限**：仅 TCP/UDP，无 HTTP 路由、无健康检查（需 Ingress）。

## 勘误 / 时效性

- ⚠️ Endpoints API 弃用、迁 **EndpointSlices**（源已注明）；userspace 模式现已移除（Linux 约 v1.26）。
- ✅ nftables "预计 v1.33 GA" 已成现实：**v1.29 alpha → v1.31 beta → v1.33 GA**（源预测准确）。

## 关联

详见实体页 [[entities/kube-proxy]] · [[entities/Service]] · [[concepts/Pod网络模型]]
