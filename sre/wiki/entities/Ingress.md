---
title: Ingress
date: 2026-06-30
tags: [kubernetes, ingress, l7, 路由, tls, ingressclass]
sources: [kubernetes/objects/ingress.md, kubernetes/objects/service.md]
---

# Ingress

Ingress 是**进入集群的 HTTP(S) 请求的 L7 路由规则集合**：在 [[entities/Service]] 之上提供对外 URL、按域名/路径路由、TLS 终止与负载均衡。它补足了 Service 的两个局限——**只有 L4、无 7 层路由**，且 NodePort/LoadBalancer 对外不灵活（见 [[entities/Service]] 的暴露层次）。

## 典型拓扑：一个 LB 入口 + L7 分流

对 HTTP(S) 流量,标准做法是**只用一个(或极少数)LoadBalancer**,引到 Ingress/Gateway 控制器,再 L7 分流:

```text
一个 LoadBalancer Service(一个公网 IP)→ Ingress/Gateway 控制器(L7)
   → 按域名/路径分流 → 各业务 Service → 真实 Pod
```

省 IP、省钱(每台云 LB+IP 都收费,见 [[entities/Service]])、还能统一 TLS/路由。**直接给单个服务配 `type: LoadBalancer` 只留给特殊场景**:非 HTTP 的 TCP/UDP 服务(Ingress 只支持 HTTP;[[entities/GatewayAPI]] 可用 TCP/UDPRoute)、内外网分离(各一个入口)、极高吞吐/特殊协议。那"一个 LB"背后是**多副本控制器 Pod**,不是单点。

## Ingress ≠ 负载均衡器：需要 Ingress Controller

Ingress 对象**只是声明的规则**，本身不创建任何 LB。集群里必须跑一个 **Ingress Controller**（以 Pod/DaemonSet 部署，watch Ingress 与 Service），它读规则去配置真正的代理（nginx、Traefik、GCE LB 等）。

> 关键区别：它**不随 [[entities/kube-controller-manager]] 自动启动**，需用户自选并部署——这与内置控制器不同。

Controller 是**一个 Pod，里面同时是"控制器"和"nginx"**，干两件事：

- **控制面（watch + 生成配置）**：watch apiserver 上的 **Ingress**（规则）、**Service/EndpointSlices**（后端 Pod IP）、[[entities/Secret]]（TLS 证书），据此**自动生成 nginx 配置并 reload**——规则不是人手写进 nginx 的。
- **数据面（nginx 代理）**：同一 Pod 里跑 nginx，**真正收流量**，按生成的配置做 L7 路由、转发到后端 Pod（常直连 EndpointSlices 的 Pod IP）。

> ⚠️ **"开 LoadBalancer Service"不是 Controller 的职责**：那个 `type: LoadBalancer` 的 Service 是你/Helm 单独建来**暴露** Controller Pod 的，背后开云 LB 的是 **cloud-controller-manager**（见 [[entities/Service]]）。Controller 既不创建也不管理它，甚至不知道自己前面有没有 LB（裸金属用 hostNetwork/NodePort 时根本没有）。（例外：GKE ingress-gce 等**云原生** Controller 会按 Ingress 去建云 LB；ingress-nginx 不会。）

## 配置示例

一个按域名 + 路径分流并终止 TLS 的 Ingress（`networking.k8s.io/v1`）：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
spec:
  ingressClassName: nginx          # 用哪个 Ingress Controller（见下 IngressClass）
  tls:                             # HTTPS：证书从 Secret 取，多 host 靠 SNI
  - hosts: ["foo.com"]
    secretName: foo-tls
  rules:
  - host: foo.com                  # 按 Host 头分流
    http:
      paths:
      - path: /api                 # foo.com/api → api-svc
        pathType: Prefix           # v1 必填：Prefix / Exact / ImplementationSpecific
        backend:
          service:
            name: api-svc
            port: { number: 80 }
      - path: /                    # 其余 → web-svc
        pathType: Prefix
        backend:
          service:
            name: web-svc          # 后端是 Service 名（不是 Pod）
            port: { number: 80 }
```

> `backend.service` 指向 [[entities/Service]]（同 namespace），不直接写 Pod；Controller 再据 EndpointSlices 找到实际 Pod。

## 如何暴露给云 LoadBalancer

Ingress 对象本身不吃流量——**吃流量的是 Ingress Controller 那些 Pod**。让外部访问到它们，就给 **Controller 自己**配一个 `type: LoadBalancer` 的 [[entities/Service]]，云厂商据此自动开一台 LB + 公网 IP：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer               # ← 云厂商据此开 LB + 分配公网 IP
  selector:
    app.kubernetes.io/name: ingress-nginx   # 选中 Controller 的 Pod
  ports:
  - { name: http,  port: 80,  targetPort: 80 }
  - { name: https, port: 443, targetPort: 443 }
```

实际很少手写这个 Service——用官方 Helm chart 装 ingress-nginx 时设 `controller.service.type=LoadBalancer` 即自动生成。步骤：

1. 装 Controller（Helm/manifest），它带上面这个 LoadBalancer Service。
2. `kubectl get svc -n ingress-nginx` 拿到 `EXTERNAL-IP`（云 LB 的公网 IP）。
3. 把域名 DNS A 记录（`foo.com`）指向该 IP。
4. `kubectl apply` 上面的 Ingress 对象，Controller watch 到后自动生成 nginx 配置。

> 云怎么"发现"这个 Service 并分配公网 IP：是 **cloud-controller-manager** 的 service controller 在 watch `type: LoadBalancer` 的 Service，调云 API 开 LB 并把 IP 写回 `status`——机制与 `<pending>`/MetalLB 详见 [[entities/Service]]「LoadBalancer 如何拿到公网 IP」。

## 路由类型

| 类型 | 规则 | 用途 |
| --- | --- | --- |
| 单服务 | 一个默认后端、无 rules | 简单暴露 / 兜底 404 |
| 多路径 | 同 host 下 `/foo`→s1、`/bar`→s2 | 按路径分发 |
| 虚拟主机 | 按 `host` 头分流、共用一个 IP | 多域名复用入口 |
| TLS | 从 [[entities/Secret]] 取 `tls.crt`/`tls.key` 做终止，多 host 靠 **SNI** 复用 | HTTPS |

## IngressClass

旧法靠 annotation `kubernetes.io/ingress.class: nginx` 指定用哪个 controller；新法用 **IngressClass** 对象（`spec.controller` 指向控制器）预声明，Ingress 经 `ingressClassName` 引用——多个 controller 并存时清晰。

## 演进：Gateway API

[[entities/GatewayAPI]] 是 Ingress 的下一代——协议更广（TCP/UDP/TLS/gRPC）、角色分离、原生扩展。新项目建议直接用 Gateway API。

> ⚠️ 勘误：源文档示例用 `extensions/v1beta1` + `serviceName/servicePort`，已过时——Ingress **v1.19 起为 `networking.k8s.io/v1`（GA）**，后端写 `backend.service.name` + `service.port.number`，路径需 `pathType`。

## 全链路：用户请求到 Pod

一个外部 HTTPS 请求打到后端 Pod 的完整路径（Ingress/Service/kube-proxy/Pod 网络四页的交汇点）：

```text
浏览器 https://foo.com/api
  │  ① 公网 DNS 解析 foo.com → 入口公网 IP
  ▼
云 LoadBalancer(L4,一个公网 IP)
  │  ② 转发到某 NodeIP:nodePort(每台 Node 同一个高位端口)
  │     (LB 后端配成 NodeIP:nodePort,是 cloud-controller-manager 读 LB Service 的
  │      spec.ports[].nodePort 后调云 API 配置的)
  ▼
NodeIP:nodePort
  │     kube-proxy 按那个 type:LoadBalancer 的 Service(selector 选中 Controller Pod、
  │     它分配了此 nodePort)的 endpoints 规则 DNAT 到某个 Controller Pod(可能在别的 Node)
  ▼
Ingress Controller Pod(nginx,L7)
  │  ③ 终止 TLS、读 Host+Path、按 Ingress 规则选中目标 Service
  │  ④ 发往该 Service 的 ClusterIP:port —— ClusterIP 是虚拟 IP、无任何实体,
  │     被 kube-proxy 预置的 iptables/ipvs 规则在内核里 DNAT 成某个 Pod IP
  │     (ingress-nginx 常直接用 EndpointSlices 选 Pod、连 ClusterIP 都不经过)
  ▼
目标 Pod IP(经 CNI 跨节点路由)
  │  ⑤ 进入 Pod 网络命名空间 → containerPort → 应用进程
  ▼
后端容器处理,响应原路返回
```

1. **公网 DNS**（非集群内 [[entities/CoreDNS]]）把 `foo.com` 解析成入口公网 IP——通常就是下一跳 LB 的地址。
2. **云 LoadBalancer**（L4）把流量送进跑在集群里的 Ingress Controller；这台 LB 由上面"暴露给云 LB"那个 `type: LoadBalancer` Service 提供，**被所有域名/服务共享**。
   - ⚠️ **"node→Controller"其实是 Service→Pod 再套一层**：LB 把流量打到某 `NodeIP:nodePort`，kube-proxy 按 Controller 那个 Service 的规则 DNAT 到某个 **Controller Pod**（默认可能在别的 Node）。**流量打到哪个 Node ≠ Controller 在哪个 Node**。
   - 两种部署决定这一跳：**Deployment + LoadBalancer Service** 常配 `externalTrafficPolicy: Local`，让 LB 只发给跑着 Controller 的 Node、免跨 Node 并保源 IP；**DaemonSet + `hostNetwork`** 则让 Controller 直接监听 Node 的 `:80/:443`，"到 Node"即"到 Controller"、无 kube-proxy 这跳（见 [[concepts/Pod网络模型]]）。
3. **Ingress Controller**（L7 分流点）：终止 TLS → 读 `Host:` 和 URL `path` → 对照 Ingress 规则选出后端 Service。L7 智能全在这一跳。
4. **Service → Pod**（L4）：经典路径发往 `ClusterIP:port`，[[entities/kube-proxy]] 的 iptables/ipvs 规则 DNAT 成某个 Pod 的 `IP:targetPort` 并做负载均衡。
5. **CNI 送达**：数据包按 Pod CIDR 跨节点路由（见 [[concepts/Pod网络模型]]），进入目标 Pod 的 network namespace，交给 `containerPort` 上的应用；容器由该节点 [[entities/kubelet]] 运行。

> ⚠️ 易错点：**很多 Controller（如 ingress-nginx）绕过 ClusterIP**——直接 watch **EndpointSlices** 拿 Pod IP 列表、自己在 Pod 间负载均衡，**不走 kube-proxy 第 ④ 跳**，少一层 DNAT 还能做 L7 会话保持。走不走 ClusterIP 取决于 Controller 实现。
>
> **DNS 参与吗**：仅在"按 Service 名代理到 ClusterIP"的经典路径下，才由 [[entities/CoreDNS]] 把 `svc.ns.svc.cluster.local` 解析成 **ClusterIP**（DNS 只给 IP，`port` 来自 Ingress 的 `backend.service.port`）；而 ingress-nginx 直连 EndpointSlices，**DNS 与 ClusterIP 一起跳过**。

## 相关

- 被路由的对象：[[entities/Service]]（L4）
- 集群外访问的几种方式对比：见 [[entities/Service]]
- 七层下一代：[[entities/GatewayAPI]]
