---

## title: kube-controller-manager

date: 2026-06-27
tags: [kubernetes, controller-manager, 控制平面, 控制器, 选主, 驱逐]
sources: [kubernetes/components/controller-manager.md]

# kube-controller-manager

Controller Manager 是 Kubernetes 的"**大脑**"——它通过 [[entities/kube-apiserver]] 监控集群状态，跑一系列**控制器（controller）**，确保现状向期望状态收敛（reconcile loop，见 [[concepts/控制平面与控制循环]]）。

它由两个进程组成：

-   **kube-controller-manager**：内置控制器集合（核心）。
-   **cloud-controller-manager（CCM）**：仅在启用 Cloud Provider 时需要，把云相关逻辑（Node、Route、Service/LB）从核心中剥离，便于云厂商在不改 K8s 核心代码的前提下扩展。

## Cloud Provider 与 CCM

**Cloud Provider** 是 K8s 与底层云平台（AWS/GCP/Azure/阿里云…）的对接层——把 K8s 意图翻译成云 API 调用。**为何单独成 CCM**：早期云代码 in-tree（编进核心），导致核心臃肿、改个云功能要等 K8s 发版、维护责任不清。拆成独立进程后——核心保持**云无关**、云厂商**按自己节奏 out-of-tree 开发**、**不改核心即可扩展**、云凭证**权限隔离**（与 CRI/CNI/CSI 同一套外置插件思路，见 [[concepts/插件机制与可扩展性]]）。启用 CCM 时，kube-controller-manager 里对应的 in-tree 控制器会被关闭以免冲突。

**关键区分：对象逻辑（K8s 自做）vs 真实基础设施（委托）**——别误以为"云上调 API、非云就核心自己干"：

| 能力                        | K8s 核心总在做                             | 需对接基础设施的部分（核心**不自做**）                                            |
| --------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------- |
| Node                        | Node 对象生命周期、NotReady 判定、Pod 驱逐 | 补云元数据/确认 VM 已删（CCM）；增删 VM（Cluster Autoscaler 调云 API）            |
| Route                       | 不自实现，委托 CNI                         | 配 VPC 路由（CCM）或 overlay 封装（CNI 插件）                                     |
| Service ClusterIP/NodePort  | **kube-proxy 全包，云无关**                | ——                                                                                |
| Service `type=LoadBalancer` | 维护 Service/endpoints                     | 开外部 LB：云上 ServiceController(CCM) 调云 API；裸金属装 MetalLB，否则 `Pending` |

> 即：**对象级逻辑永远是 K8s 自己的代码**；**真实基础设施的开通**（VM、外部 LB、云路由）核心从不自做——云上交 CCM，非云交 MetalLB/CNI/autoscaler 等插件或人工。

### 二者并行时的分工

接入云的集群里 KCM 与 CCM **同时运行**，分工：

|              | kube-controller-manager (KCM)                                               | cloud-controller-manager (CCM)                                                                             |
| ------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 跑哪些控制器 | 云无关的大部分（Deployment/RS/Job/Namespace/SA/Quota/GC/endpoints/HPA/PV…） | 仅云专属三类：Cloud Node / Route / Service(LB)                                                             |
| 对 Node      | **健康判定 + 驱逐**（NotReady/eviction）                                    | **云侧初始化 + 销毁确认**：补 zone/providerID、摘 uninitialized taint、问云"VM 还在吗"→ 已删则删 Node 对象 |
| 云凭证       | 不需要                                                                      | 单独持有（权限隔离）                                                                                       |

**怎么不打架**：KCM 与 kubelet 都带 `--cloud-provider=external` → KCM 内对应 in-tree 云控制器关闭；kubelet 注册节点时不自填云信息、先打 `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint（暂不调度）→ CCM 补全云信息后**摘除该 taint** 才可调度。这就是 kubelet 打 taint、CCM 摘 taint 的**启动接力**。

## 内置控制器（节选）

一个进程内打包了大量控制器，每个管一类对象的期望/现状收敛：

-   **工作负载**：Deployment、ReplicaSet、ReplicationController、StatefulSet、DaemonSet、Job、CronJob（对应 [[concepts/工作负载控制器]]）
-   **服务与端点**：Service、~~Endpoint~~（EndpointController 在 v1.33+ 已弃用，迁移到 EndpointSlice）
-   **节点与资源**：Node、HPA（Pod 自动扩缩）、ResourceQuota、Namespace、ServiceAccount/Token、GarbageCollector、PV/Attach-Detach、CSRSigning/Approving、TTL、Disruption

> 控制器按"必须启动 / 默认启动可选 / 默认禁用"分三组，可用 `--controllers` 选项调整。

## 高可用：Leader Election（选主）

控制器**不能多实例同时跑**（会重复创建对象），所以 HA 部署时多副本通过 `--leader-elect=true` **选主**：只有 leader 调 `StartControllers()` 跑控制器，其余副本只参与选举、待命。

-   选主基于**资源锁**：历史上用 Endpoint/ConfigMap 锁；**Endpoint API 已弃用，新版本默认/推荐 Lease 锁**。
-   这与 [[entities/etcd]] 自身的 Raft 选主是**两回事**：Raft 选 etcd 集群的 leader；这里是借 apiserver 里的一个锁对象，在多个 controller-manager 副本间选一个干活的（见 [[concepts/集群架构]] HA）。

## 高性能：Informer

v1.7 起，所有需监听资源变化的逻辑都用 **Informer**——基于事件通知的**只读本地缓存**，注册资源变化回调，极大减少对 apiserver 的直接 List/Get 调用。这是控制器规模化的关键底层机制。

**它解决什么**：几十个控制器各盯成千上万对象，若用**轮询**（反复全量 List）会压垮 apiserver/etcd 且延迟高；用**裸 Watch** 又会重复开流、没有可查询的本地视图、各自重复处理断线重连。Informer 把这些封装为：

- **一次 List 打底 + 之后只 Watch 增量**，不再反复全量拉取；
- **本地只读缓存（Indexer）**：`Get`/`List`/按 label·ns 查全走本地内存，读**不碰 apiserver**；
- **SharedInformer 共享**：同进程多控制器共用一份缓存 + 一条 Watch（而非各开一份）；
- **事件回调 → workqueue → reconcile**：事件驱动、低延迟、无轮询；
- 内建 resync / 断线重连，控制器不必各自造轮子。

**三层挡流量链**：`etcd Watch ──► apiserver(watch cache，挡住对 etcd 的重复 watch) ──► Informer(客户端缓存，挡住对 apiserver 的重复 list/watch) ──► 控制器 reconcile`。逐级放大，才让大集群里"几十控制器盯几万对象"扛得住（[[entities/etcd]] 的 Watch 是源头）。

## Node 驱逐（node eviction）

控制器最重要的容错逻辑之一——节点失联后疏散其上的 Pod：

-   kubelet 默认每 **10s**（`--node-status-update-frequency`）上报一次 Node 状态；controller-manager 每 **5s**（`--node-monitor-period`）检查一次。
-   Node 超过 **40s**（`--node-monitor-grace-period`）未更新 → 标记 **NotReady**（Ready Condition 变 `Unknown`）。
-   Node 超过 **5m** 未更新 → **驱逐其上所有 Pod**。
-   K8s 自动给 Pod 加 `node.kubernetes.io/not-ready` 和 `unreachable` 的容忍度（默认 `tolerationSeconds=300`），可被 Pod 自定义覆盖。

> **"上报/未更新"指什么**：kubelet 定期刷新 Node 的"心跳"——v1.13+ 默认续约一个**轻量 Node Lease**（`kube-node-lease` 命名空间，每 ~10s），而非每次都写完整的 NodeStatus（后者较重，仅在状态变化时写）；这是为大集群**降负载**。"超过 40s/5m 未更新" = 拿当前时间减去**最后一次心跳时间**的间隔。心跳停了通常意味着节点宕机、kubelet 挂或与 apiserver 断网——控制面分不清是哪种，故先标 `Unknown` 而非确定 down，再谨慎驱逐。

**按 Zone 限速驱逐**（防止网络分区时雪崩式误驱逐）：

> **zone 指什么**：**可用区/故障域**，由节点的拓扑标签 `topology.kubernetes.io/region` + `zone` 划分（通常由 CCM 自动打上；无标签的裸金属节点同属一个空 zone `""`）。分 zone 判断是因为故障常**按 zone 成片发生**——能区分"个别节点坏"与"整片掉线/网络分区"，避免把一个 zone 的故障误判成全局、引发驱逐风暴（与 [[concepts/集群架构]] 的"跨可用区"是同一概念）。

-   Normal（全 Ready）：默认速率 `--node-eviction-rate=0.1`（每 10s 一个节点）。
-   PartialDisruption（>33% NotReady）：小集群（<50 节点）**停止驱逐**；大集群降速到 `0.01`。
-   FullDisruption（全 NotReady）：恢复默认速率；但**所有 Zone 都 FullDisruption 时停止驱逐**（判定为是监控/网络问题而非节点真坏）。

## 相关

-   与 scheduler 的"接力"关系：[[concepts/控制平面与控制循环]]
-   它管理的各类控制器：[[concepts/工作负载控制器]]
-   选主所依赖的集群结构：[[concepts/集群架构]]
