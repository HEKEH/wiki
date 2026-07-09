---
title: NetworkPolicy（资源对象）
date: 2026-06-30
tags: [kubernetes, networkpolicy, 网络隔离, 安全, calico]
sources: [kubernetes/objects/network-policy.md]
---

# 源摘要：NetworkPolicy

feisky handbook `/concepts/objects/network-policy.md` 的要点（深入页见 [[entities/NetworkPolicy]]）。

## 核心

- 基于策略的**网络隔离**：用 label selector 模拟传统网段分割，控制 Pod 间及内外流量，减小攻击面。
- **默认全通**：不建任何策略时 Pod 之间互通；一旦某 Pod 被某策略选中，即转为**默认拒绝**、仅放行白名单（NetworkPolicy 只能"加允许"，无显式拒绝）。
- **必须有支持的 CNI 插件**（Calico、Cilium、Weave Net 等）才生效——没装则策略被忽略（见 [[concepts/插件机制与可扩展性]]）。

## 选择器与规则

- `podSelector`（选被保护的 Pod）+ `policyTypes`（Ingress/Egress）。
- 来源/去向用 `from`/`to`：`podSelector` / `namespaceSelector` / `ipBlock`（CIDR + except）。
- `ports`（protocol + port，v1.21+ 支持 `endPort` 端口范围）。
- 常用配方：default-deny、只允许特定 label Pod、禁止跨 namespace、只允许指定 namespace、允许外网。

## 版本

- v1.3 引入；**v1.7 GA（`networking.k8s.io/v1`）**；v1.8 加 Egress + ipBlock；v1.21 加 endPort。

## 不支持的场景（需服务网格/Ingress/第三方）

强制流量过网关、TLS 相关、按节点标识选目标、按服务名选、流量日志、显式拒绝、阻断 localhost/宿主访问。

## 勘误 / staleness

- 示例中 `protocol: tcp`（小写）应为 `TCP`；`kubectl run --replicas/--labels/--expose` 等旧 generator 用法已变（现 `kubectl run` 只建单 Pod）。
