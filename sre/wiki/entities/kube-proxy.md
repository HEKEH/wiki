---
title: kube-proxy
date: 2026-06-27
tags: [kubernetes, kube-proxy, service, 负载均衡, iptables, ipvs, nftables]
sources: [kubernetes/components/kube-proxy.md]
---

# kube-proxy

kube-proxy 运行在**每台 [[entities/Node]]** 上，是 [[entities/Service]] 的**数据平面实现**：它 watch [[entities/kube-apiserver]] 中 Service 与 Endpoint（现代为 EndpointSlices）的变化，在节点上配置转发规则，把发往 Service ClusterIP 的流量负载均衡到后端 Pod（仅 TCP/UDP）。

> 这回答了 home.md 的开放问题"Service 的 iptables/IPVS 转发实现"。Service 的"是什么"见 [[entities/Service]]，本页讲"怎么转发"。

部署形态：物理机进程、static Pod 或 DaemonSet。

## 先建立直觉：kube-proxy 解决什么问题

1. **Pod 的 IP 会变**：Pod 随时可能重建、漂移到别的节点，IP 跟着变,没法直接依赖（见 [[concepts/Pod网络模型]]）。
2. **Service 给一个稳定的 ClusterIP**：于是用 [[entities/Service]] 在一组 Pod 前面摆一个**固定不变的虚拟 IP**（ClusterIP）当门面。
3. **但 ClusterIP 是"假"的**：集群里**没有任何一张网卡、任何一台机器真正拥有这个 IP**——它不对应任何实体,只是个约定的"地址壳子"。
4. **所以需要有人让它生效**：必须在**每台节点**上设一套规则——"凡是发往这个 ClusterIP 的包,拦下来,改写成某个真实 Pod 的 IP 再送出去"。**干这件事的就是 kube-proxy**。

> 一句话:Service 定义了"想要一个稳定入口"这个**愿望**,kube-proxy 负责把这个愿望在每台节点上**变成真正能转发的规则**。

## 关键认知：kube-proxy 不亲自转发流量

初学者最常见的误解是"流量都流经 kube-proxy 进程"。**其实不是**:

- kube-proxy 是个**控制器**——它 watch Service/Endpoint 的变化,然后去**配置 Linux 内核的转发规则**(iptables / ipvs)。
- **真正拦截、改写、转发每个数据包的是 Linux 内核**,不是 kube-proxy 进程。kube-proxy 平时很闲,只在 Service/后端变化时去**改规则**。
- 推论:**kube-proxy 进程挂了,已有规则仍在内核里、流量照常转发**;只是这期间**新建/变更的 Service 不会被更新**而已。

```text
         设置规则                    内核按规则转发
kube-proxy ───────► iptables/ipvs 规则 ◄═══════ 数据包（Pod→ClusterIP→真实 Pod）
（控制面，平时闲）     （内核，真正干活）
```

## 一次访问的完整过程（ClusterIP 例子）

设 `nginx` 是个 ClusterIP Service(ClusterIP=`10.96.0.66`,后端 3 个 nginx Pod),Pod A 访问 `http://nginx`:

1. **DNS 解析**:[[entities/CoreDNS]] 把 `nginx` 解析成 ClusterIP `10.96.0.66`。
2. **发包**:Pod A 把请求发往 `10.96.0.66:80`。
3. **内核拦截 + 改写**:包在内核网络栈被 kube-proxy 预先设好的规则拦下,从 3 个后端里**随机挑一个**,把**目标地址 DNAT 改写**成某个真实 Pod IP(如 `10.244.2.5:80`)。
4. **送达**:包经 Pod 网络(CNI)送到那个 nginx Pod。
5. **回包**:靠内核的**连接跟踪(conntrack)** 原路把地址改回,对 Pod A 来说,自始至终就像在和 `10.96.0.66` 通信——它根本不知道背后是哪个 Pod。

### 术语:DNAT / 回包 / conntrack

包头有**源地址**和**目标地址**(各为 `IP:端口`),NAT 就是在转发途中改写其一:

- **DNAT**(改**目标**):去程把目标从虚拟 ClusterIP 换成真实 Pod IP——这就是"转发到后端"的实质动作。例:目标 `10.96.0.66:80` → `10.244.2.5:80`。
- **回包为何要改回**:一条 TCP 连接由**四元组**(源IP/源端口/目标IP/目标端口)唯一标识。Pod A 当初发给 `10.96.0.66`,只认"来自 `10.96.0.66:80` 的回复";若 nginx Pod 直接以源 `10.244.2.5` 回,四元组对不上,**Pod A 会丢弃**。故回包的源必须改回 `10.96.0.66:80`。
- **conntrack 自动逆转**:内核做 DNAT 时在**连接跟踪表**记一笔,回包属同一连接 → 内核**自动套用反向翻译**把源改回 ClusterIP。**只需配去程一条规则,回程自动逆转**,客户端全程"以为"只在和 ClusterIP 通信。
- **SNAT/MASQUERADE**(改**源**):有时去程还改源地址(如跨节点),确保回包先回到做过 DNAT 的节点,conntrack 才有记录可逆转。

> 下面的 iptables 链就是在编排这些 DNAT/SNAT 改写动作。

## 代理模式（proxy mode）

| 模式 | 引入 | 原理 | 主要问题 / 特点 |
| --- | --- | --- | --- |
| **userspace** | 最早 | 用户态进程监听端口，iptables 把流量转给它，再由它转发到 Pod | 流量绕用户态，效率低、性能瓶颈（已废弃）|
| **iptables** | 长期默认 | 纯 iptables 规则做 DNAT 转发 | 服务多时规则爆炸、**非增量更新**有时延，大规模性能差 |
| **ipvs** | v1.8 测试 / v1.11 GA | 基于内核 IPVS（LVS），**增量更新**、service 更新期间连接不断 | 解决 iptables 规模问题；需预加载 `ip_vs*`、`nf_conntrack` 内核模块 |
| **nftables** | v1.29 alpha / v1.31 beta / **v1.33 GA** | "verdict map" 路由，包处理从 O(n) 降到 ~O(1)，消除全局锁竞争 | 显著降低大集群首包延迟；需内核 **5.13+**，与部分网络组件不完全兼容 |
| **winuserspace** | — | 同 userspace，仅 Windows | — |

> 源文档"预计 v1.33 GA"已成现实——nftables 模式 **v1.29 alpha → v1.31 beta → v1.33 GA**，截至当前（v1.33+）已 GA。userspace 模式则已移除（Linux 约 v1.26）。未来 nftables 可能取代 ipvs，iptables 仍长期保留。

## iptables 模式怎么转发

### 先懂三个 iptables 概念

- **规则(rule)**：一条"**若包满足条件 X，就执行动作 Y**"(如"目标是 10.96.0.66:80 → DNAT 到某 Pod")。
- **链(chain)**：一串**按顺序排列的规则**；包进来后**从上往下逐条匹配**，命中即按其动作处理。
- **跳转(jump,`-j`)**：一条规则的动作可以是"**跳到另一条链继续处理**"，类似函数调用——借此把规则**分组成命名子链**。kube-proxy 建的子链都以 `KUBE-` 开头。

> 类比电话总机的多级菜单：总台按"找哪个服务"转接 → 二级"随机分一个坐席" → 三级"接通该坐席的分机"。

### 三级链走一遍（nginx 例子：ClusterIP `10.96.0.66:80`，3 个后端）

**① `KUBE-SERVICES`——总入口,按"找哪个 Service"分发**

```text
目标 == 10.96.0.66:80（nginx 的 ClusterIP）→ 跳到 KUBE-SVC-NGINX
目标 == 其它 ClusterIP                      → 跳到对应的 KUBE-SVC-...
```

**② `KUBE-SVC-NGINX`——该 Service 的负载均衡,随机挑一个后端**

```text
以 1/3 概率 → KUBE-SEP-1（后端A）
以 1/2 概率 → KUBE-SEP-2（后端B）
否则        → KUBE-SEP-3（后端C）
```

为什么是 `1/3、1/2、否则`？因规则**顺序执行、每条只作用于上一条"漏下来"的包**：第1条取走 1/3，剩 2/3；第2条对这 2/3 取 1/2 = 总量 1/3，剩 1/3；第3条全收。**最终 A/B/C 各 1/3,均匀分流**(原始规则里 `0.333`、`0.5` 即此)。

**③ `KUBE-SEP-1/2/3`——具体后端,真正做 DNAT 改写目标**(SEP = Service EndPoint,每后端一条)

```text
DNAT --to-destination 10.244.2.5:80   ← 把目标从 ClusterIP 换成真实 Pod
```

**串起来**：`包 → KUBE-SERVICES(找 nginx) → KUBE-SVC-NGINX(掷骰子选 B) → KUBE-SEP-2(目标改写成 10.244.2.7:80)` → 交 CNI 送达。

### 辅助链

- **`KUBE-MARK-MASQ` / `KUBE-POSTROUTING`**：给需要的包打标记,出站做 **SNAT/MASQUERADE**(改源地址)——即前面"回包找回路"那套,跨节点时尤其需要。
- **`KUBE-NODEPORTS`**：处理从节点端口进来的 **NodePort** 流量,匹配后同样跳进 `KUBE-SVC-...` 走负载均衡。
- **`KUBE-FW-xxx`**：处理 **LoadBalancer** 的外部入口 IP。
- **`externalTrafficPolicy: Local`**：若开启且**本节点无该 Service 的 Pod**,**直接丢弃**外部来包——让外部流量只落到真有后端的节点,从而**保留客户端真实源 IP**(免去跨节点转发与 SNAT)。

## ipvs 模式

转发由内核 IPVS 完成（`ipvsadm -ln` 可查虚拟服务与后端权重），但**仍用少量 iptables + ipset** 做 SNAT/伪装和规则归类（如 `KUBE-CLUSTER-IP`、`KUBE-NODE-PORT-TCP` 等 ipset），避免 iptables 规则随服务数线性膨胀。

## 局限

只支持 TCP/UDP、**不支持 HTTP 路由、无健康检查**——七层路由需 Ingress Controller 解决。

## 相关

- Service 抽象与 ClusterIP/NodePort：[[entities/Service]]
- Pod IP 从哪来：[[concepts/Pod网络模型]]
- 同级节点组件：[[entities/Node]]、[[entities/kubelet]]
