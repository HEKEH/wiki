---
title: Service
date: 2026-06-23
tags: [kubernetes, service, 服务发现, 负载均衡, endpointslices]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes 101.md, kubernetes/objects/service.md]
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

四种类型，**层层叠加**（后者在前者基础上扩展）：

| 类型 | 含义 | 对外 |
| --- | --- | --- |
| **ClusterIP**（默认） | 仅集群内可达的虚拟 IP（如 `http://10.0.0.66`） | 否 |
| **NodePort** | 在每台 Node 开一个端口，`<NodeIP>:NodePort` 可达 | 集群内外 |
| **LoadBalancer** | 在 NodePort 上叠 **cloud provider 的外部 LB**；物理机可用 MetalLB | 公网 |
| **ExternalName** | 不分配 ClusterIP，用 **DNS CNAME** 把服务名转发到外部域名 | — |

> 暴露层次：**Service 是 L4**。要 L7（按域名/路径路由、TLS）用 [[entities/Ingress]] 或其下一代 [[entities/GatewayAPI]]；集群外访问还可用 NodePort、LoadBalancer、Ingress 或把 ClusterIP 网段路由出去（ECMP）。

### LoadBalancer 如何拿到公网 IP

不是云"主动发现"，而是集群里的 **cloud-controller-manager（CCM）** 的 **service controller** 在 **watch `type: LoadBalancer` 的 Service**（套用 [[concepts/控制平面与控制循环]] 的 reconcile 范式）：

1. `apply` 一个 `type: LoadBalancer` 的 Service。
2. CCM watch 到 → 调**云厂商 API** 开一台真实云 L4 LB（AWS NLB/ELB、GCP forwarding rule、阿里 SLB…）。
3. 云返回公网 IP → CCM 写回该 Service 的 **`status.loadBalancer.ingress`**，即 `kubectl get svc` 里的 **`EXTERNAL-IP`**。

- **前提**：集群跑着 CCM 且配了云凭证。托管集群（EKS/GKE/AKS）开箱即用；**自建且无云 provider 时 `EXTERNAL-IP` 永远 `<pending>`**——裸金属用 **MetalLB** 扮演该角色从本地 IP 池分配。
- **底层链路**：`type: LoadBalancer` 同时隐含分配一个 NodePort，云 LB 只把流量打到**节点的 NodePort**，到 Pod 的最后一跳仍是 [[entities/kube-proxy]]：`公网 → 云 LB → NodeIP:nodePort → kube-proxy DNAT → Pod`。配 `externalTrafficPolicy: Local` 可省一跳并保留源 IP（见下）。
- **一个 Service 默认一个外部 IP**；**端口不是分配的、是你 `spec.ports[].port` 声明的**(同 IP 可开多端口)。每台云 LB + IP 通常都**产生费用**——所以**不给每个服务都配 LoadBalancer**，而是用 [[entities/Ingress]]/[[entities/GatewayAPI]] 让多服务**共享一个 LB 入口**(L7 按域名/路径分流)。可共享/指定 IP:MetalLB 的 `allow-shared-ip` 注解、或(已弃用的)`spec.loadBalancerIP`。

### 三个端口：port / nodePort / targetPort

NodePort/LoadBalancer 型 Service 实际涉及**三个端口**，别混淆（以 `port:80 → targetPort:8080` 为例）：

| 字段 | 谁监听 | 说明 |
| --- | --- | --- |
| **`port`** | ClusterIP（及云 LB 对外） | Service 前端口；集群内 `ClusterIP:port`、云 LB 对外也用它 |
| **`nodePort`** | 每台 Node | 自动分配（默认 30000–32767，`--service-node-port-range` 可改），**云 LB 实际转发到这里**；不写则系统分配，非 80 这类低端口 |
| **`targetPort`** | 后端 Pod 容器 | 流量最终落到的 `containerPort` |

- **`port` 与 `targetPort` 不必相等**（前者对外、后者是容器实际端口）；数字相同只是好记。
- **`nodePort` 全集群统一**：一个 Service（每个端口）只分配**一个** nodePort 号，kube-proxy 在**所有 Node** 上都开这同一个号，故任意 `NodeIP:nodePort` 皆可入，且全局唯一、别的 Service 不能复用。
- 完整端口链：`客户端:port → 云 LB:port → NodeIP:nodePort(高位) → kube-proxy DNAT → PodIP:targetPort`。

## Headless 与无 selector 的特殊形态

- **Headless Service（`clusterIP: None`）**：不分配 ClusterIP，DNS 直接返回**后端 Pod 的 A 记录列表**（客户端自己选/做有状态寻址）——[[entities/StatefulSet]] 靠它给每个 Pod 固定 DNS（见 [[entities/CoreDNS]]）。
- **无 selector 的 Service**：不写 selector，手动建同名 endpoint（推荐 EndpointSlice）指向**集群外 IP**，从而把外部服务（如外部数据库）也包装成 Service。

## 源 IP 与流量策略

- ClusterIP 内部流量**不 SNAT**，后端能看到真实源；NodePort/LoadBalancer 默认 **SNAT**（后端只看到 Node IP）。
- **两级负载均衡**：云 LB 在**多个 Node 间**分发（第一级），kube-proxy 再在**多个 Pod 间**分发（第二级）。默认 `externalTrafficPolicy: Cluster` 下云 LB 把**所有 Node**纳入后端池、流量可落任意 Node，本地无 Pod 时 kube-proxy 跨 Node 转发（多一跳 + SNAT，故丢源 IP）。
- `externalTrafficPolicy: Local`：云 LB 健康检查**只在有本 Service Pod 的 Node 通过**，故只把流量发给这些 Node、kube-proxy 只转本地 Pod（本地无则丢包），从而**保留客户端真实源 IP**、省一跳；代价是 Pod 在各 Node 分布不均时**负载可能不均**（细节见 [[entities/kube-proxy]]）。
- `internalTrafficPolicy: Local`：集群内部流量也只发本节点 endpoint。

## endpoints 的现代实现：EndpointSlices

后端 `IP:端口` 列表早期记在单个 **Endpoints** 对象里；规模大时它臃肿。现用 **EndpointSlices**（`discovery.k8s.io/v1`）：一个 Service 对应**多个 slice**（按 label `kubernetes.io/service-name` 关联），支持双栈、扩展性更好。

> ⚠️ **Endpoints API 于 v1.33 弃用**（仍保留不移除，但新功能只进 EndpointSlices）。EndpointSlice 由 controller-manager 的 EndpointSlice 控制器维护（见 [[entities/kube-controller-manager]]）。
> 较新：**多 Service CIDR（v1.33 GA）** 经 `ServiceCIDR`/`IPAddress` 对象可给 ClusterIP 动态加多个网段。

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
- L7 路由与对外暴露：[[entities/Ingress]]、[[entities/GatewayAPI]]。
- 限制 Pod 间访问：[[entities/NetworkPolicy]]。
