---
title: Gateway API
date: 2026-06-30
tags: [kubernetes, gateway-api, l7, 路由, ingress演进, 角色分离]
sources: [kubernetes/objects/gateway-api.md, kubernetes/objects/ingress.md]
---

# Gateway API

Gateway API 是 [[entities/Ingress]] 的**下一代演进**（SIG-NETWORK 维护）：表达力更强、可扩展、**面向角色**。它针对 Ingress 三大局限——只能简单 HTTP 路由、功能靠 controller 私有注解、无角色/权限分离。

## Ingress 三大局限

1. **能力弱、协议单一**：原生只有"host + path"转发、**仅 HTTP/HTTPS**；按 header/method/cookie 路由、流量切分、改写/镜像，以及 TCP/UDP/gRPC/TLS-passthrough 都不支持 → Gateway API 用丰富 match/filter 的 **HTTPRoute** + **TCP/UDP/TLS/GRPCRoute** 覆盖。
2. **靠 controller 私有注解补功能**：额外能力塞进 `nginx.ingress.kubernetes.io/*` 这类注解——**换 controller 就失效、字符串值无 schema 校验、各家语义不一** → Gateway API 提升为**原生 API 字段/CRD**（跨实现统一、有校验）。
3. **无角色/权限分离**：一个 Ingress 对象糅合了入口/证书/端口（平台运维关心）与路由规则（应用开发关心），[[concepts/RBAC]] 只能按"整个对象"授权 → Gateway API 拆成 **GatewayClass / Gateway / HTTPRoute** 三层对应三类人，并经 `parentRefs` 做**受控委托**（见下）。

## 三个核心资源 + 角色分离

按"谁负责什么"拆成三层资源，对应三类人：

| 资源 | 作用 | 谁管 |
| --- | --- | --- |
| **GatewayClass** | 声明一类网关 + 控制器（类比 [[concepts/Volume存储]] 的 StorageClass） | 基础设施提供者 |
| **Gateway** | 定义 listeners：端口/协议/hostname | 集群操作员 |
| **HTTPRoute** | `parentRefs` 挂到 Gateway，按 host/path 路由到 `backendRefs`（Service） | 应用开发者 |

> 这种分离让平台团队管入口、应用团队管路由，权限边界清晰——正是 Ingress 缺的。

路由资源除 HTTPRoute 外还有 **TLSRoute / TCPRoute / UDPRoute / GRPCRoute**——协议覆盖远超 Ingress 的仅 HTTP(S)。

## 用法示例（三对象串起来）

```yaml
# ① GatewayClass（集群级，基础设施提供者）：选用哪套实现——通常装实现时自带，直接引用即可
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata: { name: eg }
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller   # 由哪个控制器实现（如 Envoy Gateway）
---
# ② Gateway（infra ns，集群运维）：入口端口/协议 + 允许哪些 Route 挂
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: web-gateway, namespace: infra }
spec:
  gatewayClassName: eg              # 引用实现自带的 GatewayClass（① 基础设施提供者管）
  listeners:
  - { name: http, protocol: HTTP, port: 80,
      allowedRoutes: { namespaces: { from: Selector, selector: { matchLabels: { gateway-access: "true" } } } } }
---
# ③ HTTPRoute（team-a ns，应用开发者）：挂到 Gateway，按 host/path 路由到 Service
#    前提：team-a 这个 ns 须带 label gateway-access=true，才符合上面 Gateway 的 allowedRoutes、准许挂靠
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api-route, namespace: team-a }
spec:
  parentRefs: [{ name: web-gateway, namespace: infra }]   # 主动"挂靠"到 Gateway
  hostnames: ["foo.com"]
  rules:
  - matches: [{ path: { type: PathPrefix, value: /api } }]
    backendRefs: [{ name: api-svc, port: 80 }]            # 可列多个 + weight 做金丝雀
```

串接方向:**HTTPRoute 用 `parentRefs` 主动挂 Gateway,Gateway 用 `allowedRoutes` 决定准不准挂**(受控委托);`gatewayClassName` 选实现。对比 [[entities/Ingress]] 一个对象塞入口+路由,这里拆成入口(Gateway)与路由(HTTPRoute)两个对象、分属两个团队。

### 怎么生效与验证

用 **`kubectl apply -f`**（声明式、幂等，改了再 apply 即更新）。三对象可放同一文件、一条命令下发,但**前置与顺序**要对:

```bash
# 1) 先装 CRD —— 否则 apply 会报 "no matches for kind ...gateway.networking.k8s.io"（kind 不存在）
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
# 2) 装一种实现（带控制器 + GatewayClass，如 Envoy Gateway，helm/manifest）
# 3) 建 namespace 并给 team-a 打准入 label（否则 Route 挂不上，见上）
kubectl create ns infra && kubectl create ns team-a
kubectl label ns team-a gateway-access=true
# 4) 下发三对象
kubectl apply -f gateway-setup.yaml
```

- **CRD 必须先于 CR**：apiserver 得先认识这些 kind（这正是"非内置、要装 CRD"的后果，见下）。
- **对象间顺序不敏感**：Gateway/HTTPRoute 谁先都行，`parentRefs` 引用在两边都存在后由控制器自动解析；但 namespace 与 label 要先就绪。
- **确认真正生效**（看 status，不是 apply 成功就行）：`kubectl get gateway -n infra`（看 `ADDRESS`/`PROGRAMMED=True`）、`kubectl describe httproute api-route -n team-a`（看 conditions:`Accepted`=挂上 Gateway、`ResolvedRefs`=后端 Service 找到）。

## 权限隔离（两层叠加）

角色分离要落地靠两层机制配合:

- **控制面授权 = [[concepts/RBAC]]**:按"资源类型 + namespace"分授——GatewayClass 是**集群级**用 `ClusterRole`(平台团队)、Gateway 在 `infra` ns 用 `Role`(运维)、HTTPRoute 在各自 ns 用 `Role`(应用团队)。于是应用团队只能改自己 ns 的 Route,**碰不到 Gateway/GatewayClass**——这正是 Ingress 单对象做不到的分权。
- **数据面委托 = Gateway API 自带**:光有 RBAC 不够——Route 能否**挂上**共享 Gateway 由 Gateway 的 **`allowedRoutes`**(平台准入)说了算;HTTPRoute 若要**跨 namespace 引用** Service,目标 ns 还须建 **`ReferenceGrant`** 显式放行(默认拒绝跨 ns 引用)。

> 一句话:**RBAC 管"谁能增删改哪类对象",`allowedRoutes`/`ReferenceGrant` 管"哪些 Route 准挂入口、能否跨 ns 引用",二者叠加实现真正的角色隔离。**

## vs Ingress

| | Ingress | Gateway API |
| --- | --- | --- |
| 协议 | 仅 HTTP/HTTPS | HTTP/TCP/UDP/TLS/gRPC |
| 角色分离 | 无 | 三层清晰分离 |
| 扩展方式 | controller 私有注解 | 原生 API 扩展 |

## v1.3.0（2025-04）特性举要

- **按百分比请求镜像**（蓝绿/压测，标准通道）。
- 实验通道：**CORS 过滤**、**XListenerSets**（跨 namespace 合并监听器）、**重试预算（XBackendTrafficPolicy）**、**Inference Extension**（`InferencePool`/`InferenceModel`，为 LLM 推理做模型感知路由 + GPU 优化负载均衡）。

## 是否自带 / 谁来实现

- **API 定义:官方但非内置**。Gateway API 由 K8s 的 **SIG-Network** 维护(`kubernetes-sigs/gateway-api`),但**不打包进核心**——以 [[entities/CustomResourceDefinition]] 形式**单独装 CRD**(故意做成 CRD 以便独立演进)。对比 [[entities/Ingress]] 的 API **内置**于 `networking.k8s.io/v1`,这是两者最大交付差异。
- **实现:一律第三方**。[[entities/kube-controller-manager]] 里**没有** Gateway 控制器,K8s 核心不提供任何实现;须自选一种并部署——Envoy Gateway、Istio、Cilium、NGINX Gateway Fabric、Kong,或云厂商(GKE Gateway)等,每种自带一个 GatewayClass。这点与 Ingress 相同(API 是契约、控制器靠生态)。

## 兼容性

要求 **K8s 1.26+**；标准通道已达 **v1 稳定**；实现有 Envoy Gateway、Istio、Cilium、Airlock 等（需安装 CRD + 选一种实现）。

## 相关

- 上一代：[[entities/Ingress]]
- 路由目标：[[entities/Service]]
