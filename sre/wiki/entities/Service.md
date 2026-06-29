---
title: Service
date: 2026-06-23
tags: [kubernetes, service, 服务发现, 负载均衡]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes 101.md]
---

# Service

Service 是**应用服务的抽象**，为一组 [[entities/Pod]] 提供统一入口、负载均衡和服务发现。它解决了 Pod IP 随重启变化、不能直接依赖的问题（Pod IP 的由来见 [[concepts/Pod网络模型]]）。

## 工作原理

- 通过 [[concepts/Label与Selector]] 选中一组 Pod，这些 Pod 的 `IP:端口` 列表组成 **endpoints**。
- 由 **[[entities/kube-proxy]]** 负责将 Service 的访问流量负载均衡到这些 endpoints 上（iptables/ipvs/nftables 等数据面实现详见该页）。
- 每个 Service 自动分配一个 **cluster IP**（仅集群内部可访问的虚拟地址）和 **DNS 名**，其他容器通过该地址/DNS 访问，无需了解后端 Pod。

## 定义示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 8078        # service 对外暴露的端口
    name: http
    targetPort: 80    # 后端容器端口
    protocol: TCP
  selector:
    app: nginx
```

## 端口与 targetPort

`port` 是 Service 对外暴露的端口，`targetPort` 是流量转发到的**后端 Pod 端口**。关于"多个后端的端口必须一样吗"：

### targetPort：数字 vs 命名端口

- **写数字**（如 `targetPort: 80`）：该端口号对**所有**被选中的 Pod 统一生效——要求所有副本都监听同一端口号。
- **写名字**（命名端口）：`targetPort` 可指向 Pod 中 `containerPort` 的 `name`，**各 Pod 的实际端口号可以不同**，靠名字对齐。适合滚动升级、不同版本监听端口不一致的场景。

```yaml
# Service：targetPort 用名字
ports:
- port: 8078
  targetPort: web
---
# Pod A 监听 80、Pod B 监听 8080，但都把端口命名为 web
containers:
- ports:
  - containerPort: 80      # Pod A
    name: web
# （Pod B 则 containerPort: 8080, name: web）
```

> 一个 Pod 内多容器**共享同一网络 namespace 和 Pod IP**，故某端口在 Pod 内只能被一个容器占用；Service 的 targetPort 指向"Pod IP 上的端口"，不挑具体容器——谁 bind 了谁收到。见 [[entities/Pod]]。

### 一个 Service 可暴露多个端口

`ports` 是列表，可配多条（多条时每条须带 `name`）：

```yaml
ports:
- name: http
  port: 80
  targetPort: 8080
- name: metrics
  port: 9090
  targetPort: 9090
```

## 服务类型与暴露

通过 `kubectl expose` 可快速创建：

```bash
kubectl expose deployment nginx-app --port=80 --target-port=80 --type=NodePort
```

- **ClusterIP**（默认）：仅集群内可访问，如 `http://10.0.0.66`。
- **NodePort**：在每个 Node 上开放一个端口（如 30772），集群内外均可通过 `http://<node-ip>:30772` 访问。

> 例：ClusterIP 服务在集群内可用 `http://10.0.0.66` 访问；NodePort 则集群外也能用 `http://<node-ip>:30772` 访问。

## DNS 名与命名规则

每个 Service 自动获得一个集群内 DNS 名，完整形式：

```text
<service>.<namespace>.svc.cluster.local
```

例如 `team-b` 命名空间里的 `api` 服务：

```bash
curl http://api                          # 同 namespace 内可省略 namespace
curl http://api.team-b                   # 跨 namespace 简写
curl http://api.team-b.svc.cluster.local # 全称
```

### 为什么是 `<service>.<namespace>` 而非 `<namespace>.<service>`

虽然 service 隶属于 [[entities/Namespace]]，但这里复用的是 **DNS 命名约定：最具体的在最左、范围逐级向右变大**（与文件路径"父在前"相反）。和普通域名 `www.google.com` 同理：

```text
api  .  team-b  .  svc  .  cluster.local
服务     namespace  资源类型   集群根域
具体 ←──────────────────────→ 笼统
```

从左往右正是"层层向上"：`api`（最具体的服务）→ `team-b`（它所在的 namespace）→ `svc`（service 类资源）→ `cluster.local`（集群根域）。所以 `api.team-b` 读作"team-b 里的 api"，隶属关系与直觉一致，只是 DNS 用"子在左、父在右"来表达。

> 跨 namespace 默认网络互通，正是靠这套 DNS 名访问，详见 [[entities/Namespace]] 的隔离边界说明。

## 相关

- 数据面如何转发（iptables/ipvs/nftables）：[[entities/kube-proxy]]。
- DNS 名怎么解析（CoreDNS、A/SRV 记录、Headless）：[[entities/CoreDNS]]。
- 自动伸缩/回收的 Pod 会自动加入/移出 Service 的 endpoints，见 [[concepts/扩缩容与滚动升级]]。
- [[concepts/健康检查]] 中的 ReadinessProbe 决定 Pod 是否接收 Service 流量。
