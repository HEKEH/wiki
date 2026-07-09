---
title: NetworkPolicy
date: 2026-06-30
tags: [kubernetes, networkpolicy, 网络隔离, 安全, 微隔离]
sources: [kubernetes/objects/network-policy.md]
---

# NetworkPolicy

NetworkPolicy 是 K8s 的**网络隔离/微隔离**对象：用 [[concepts/Label与Selector]] 模拟传统网段分割，控制 [[entities/Pod]] 之间以及与外部的流量，缩小攻击面。

## 两条核心规则

1. **默认全通**：不建任何策略时，集群内 Pod 互通（见 [[concepts/Pod网络模型]]"每 Pod 一 IP、默认互联"）。
2. **被选中即默认拒绝**：一旦某 Pod 被任一 NetworkPolicy 的 `podSelector` 选中，它在对应方向（Ingress/Egress）就**只放行白名单**、其余全拒。NetworkPolicy **只能"加允许"，没有显式拒绝**——"拒绝"是靠"选中但不放行"达成的。

## 必须有支持的 CNI 插件

NetworkPolicy 是**声明**，真正执行靠网络插件（**Calico、Cilium、Weave Net** 等）。**用的 CNI 不支持就静默失效**（如默认 flannel 不实施）——这是常见踩坑点（见 [[concepts/插件机制与可扩展性]]）。

## 规则构成

```yaml
spec:
  podSelector: { matchLabels: { role: db } }   # 保护谁
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: { matchLabels: { role: frontend } }   # 哪些 Pod
    - namespaceSelector: { matchLabels: { project: x } } # 哪些 namespace
    - ipBlock: { cidr: 172.17.0.0/16, except: [172.17.1.0/24] }  # 哪些网段
    ports: [{ protocol: TCP, port: 6379 }]
```

- 来源/去向三选器：`podSelector` / `namespaceSelector` / `ipBlock`。
- 常用配方：**default-deny**（`podSelector: {}` + policyTypes）、只允许特定 label、禁止/只允许跨 namespace、允许外网。
- `endPort`（v1.21+）支持端口范围。

## 配方：各 namespace 默认互相隔离

NetworkPolicy 是 **namespaced 对象、无集群级默认**，所以"默认跨 ns 隔离"= 在**每个 namespace** 放一条"只放行本 ns"的策略：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-cross-namespace, namespace: team-a }  # 每个 ns 各一份
spec:
  podSelector: {}             # 选中本 ns 所有 Pod → 触发默认拒绝
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {}         # 空 podSelector 且无 namespaceSelector = 仅"本 ns 内 Pod"
```

效果：**同 ns 内互通、跨 ns 一律拒**。让它覆盖所有（含新建）ns 的三种办法：

- **GitOps 批量**：Helm/Kustomize/ArgoCD 给每个 ns 各下发一份。
- **自动注入（推荐）**：**Kyverno**/Gatekeeper 的 generate 规则，新建 ns 时**自动创建**这条策略。
- **CNI 集群级策略**：**Calico `GlobalNetworkPolicy`**、**Cilium `CiliumClusterwideNetworkPolicy`** 一条覆盖全局，免逐 ns 复制。

必留的放行口子（否则集群异常）：**DNS**（到 kube-system 的 53，尤其加了 Egress 时）、**Ingress 控制器**（`ingress-nginx` ns → frontend）、**监控**（Prometheus 抓 metrics）。跨 ns 放行用 `namespaceSelector`：

```yaml
  ingress:
  - from:
    - podSelector: {}                                            # 本 ns
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: monitoring } }  # 额外放行监控 ns
```

> 同类三层配方 `frontend→backend→db`：策略写在**目的方**（"谁能进我"）——backend 只放行 `tier=frontend`、db 只放行 `tier=backend`，再加 default-deny 兜底。

## 做不到的事（需服务网格 / Ingress / 第三方）

强制流量过统一网关、TLS 层策略、按节点标识选目标、按服务名选、生成流量日志、显式拒绝、阻断 localhost/宿主访问。

> 版本：v1.3 引入，**v1.7 GA（`networking.k8s.io/v1`）**，v1.8 加 Egress + ipBlock。

## 相关

- 网络模型与"默认互通"前提：[[concepts/Pod网络模型]]
- namespace 边界：[[entities/Namespace]]
- 选择器语法：[[concepts/Label与Selector]]
