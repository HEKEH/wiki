---
title: Pod 网络模型
date: 2026-06-23
tags: [kubernetes, 网络, pod, cni, ip, namespace]
sources: [kubernetes/kubernetes基本概念.md]
---

# Pod 网络模型

Kubernetes 网络最基本的一条原则是 **"每个 [[entities/Pod]] 一个独立 IP"**（IP-per-Pod）。理解这条，就能解释"为什么同机的 Pod 也各有 IP""为什么多个 Pod 都听 80 也不冲突"等一系列问题。

## 核心：每个 Pod 一个独立 IP

- 每个 Pod 拥有自己独立的 **network namespace**（Linux 内核隔离机制，见 [[concepts/容器]]）——独立的网卡、IP、端口空间、路由表。
- **与是否同机无关**：哪怕 Pod A、Pod B 在同一台 [[entities/Node]] 上，它们也各自拿到不同的 IP。
- **Pod 内多容器共享**这一个网络 namespace 和 IP，所以同 Pod 内容器可用 `localhost` 互访，但某端口在 Pod 内只能被一个容器占用。

```text
        一台 Node
   ┌──────────────────────────────┐
   │  Pod A  (netns A)  10.1.0.3:80 │
   │  Pod B  (netns B)  10.1.0.7:80 │   ← 同机，不同 IP，互不冲突
   └──────────────────────────────┘
```

## IP 从哪来：CNI 插件

IP 分配由 **CNI（Container Network Interface）插件**完成（见 [[concepts/插件机制与可扩展性]]）：

1. 每台 Node 分到一段子网（**Pod CIDR**），如 `10.1.0.0/24`。
2. 每起一个 Pod，kubelet 调用 CNI 插件：
   - 为 Pod 创建独立的 network namespace；
   - 造一对**虚拟网卡（veth pair）**：一端进 Pod 成为其 `eth0`，另一端接到宿主机网桥/路由；
   - 从该 Node 的子网里分配一个 IP 给 Pod。
3. 于是同一 Node 上的各 Pod 从这段子网拿到**不同 IP**。

> 类比虚拟机：一台物理机能跑多个有独立 IP 的虚拟机；namespace 让一台机器上的多个 Pod 达到类似效果，但远比虚拟机轻量（只隔离网络等视图，不虚拟整个 OS）。

## 端口冲突：只看"是否共享网络 namespace"

| 范围 | 端口能否重复 | 原因 |
| --- | --- | --- |
| 同一 Pod 内多容器 | ❌ 不能 | 共享同一 network namespace 与 IP |
| 不同 Pod 之间（含同机） | ✅ 可以，且常见 | 各有独立 namespace 与 IP |

例外——**占用 Node IP 的端口会在同一 Node 上冲突**：

- **`hostPort`**：容器端口映射到所在 Node 的端口。两个用 `hostPort: 80` 的 Pod 调度到同一 Node 会冲突，导致其一起不来（故一台 Node 只能放一个）。
- **NodePort 类型 Service**：在每台 Node 开一个端口（如 30772），Node 级唯一，由 K8s 自动分配，一般不必担心冲突。

> 关键区分：`containerPort` 占的是 **Pod IP**（Pod 间天然隔离，不冲突）；`hostPort` / `NodePort` 占的是 **Node IP**（同 Node 有限，可能冲突）。

## 访问与隔离

- **访问**：Pod IP 会随重启变化，不直接依赖；通过 [[entities/Service]] 的 cluster IP / DNS 名稳定访问。
- **隔离**：默认所有 Pod（跨 [[entities/Namespace]] 也）网络互通；要限制需用 **[[entities/NetworkPolicy]]**，且依赖支持它的 CNI 插件。

## 验证

```bash
kubectl get pods -o wide   # 同一 NODE 列下，各 Pod 的 IP 互不相同
```

在 Node 上也可用 `lsns -t net` / `ls -l /proc/<pid>/ns/net` 查看为各 Pod 创建的独立网络 namespace。

## 相关

- 隔离机制底层：[[concepts/容器]]（namespace）
- IP 分配者：[[concepts/插件机制与可扩展性]]（CNI）
- 稳定访问入口：[[entities/Service]]
- 跨 namespace 互通与隔离：[[entities/Namespace]]
