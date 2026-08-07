---
title: ResourceQuota / LimitRange
date: 2026-06-30
tags: [kubernetes, resourcequota, limitrange, 配额, 多租户]
sources: [kubernetes/objects/quota.md]
---

# ResourceQuota / LimitRange

这两个对象一起做 [[entities/Namespace]] 级的**资源治理**——多租户/多团队共享集群时防止某个 namespace 吃光资源。

## ResourceQuota：限 namespace 总用量

- 作用于单个 namespace（**每 namespace 最多一个**），靠准入控制 `ResourceQuota` 实施；**超额即拒绝**新建资源。
- 开启计算配额后，新建容器**必须**声明 requests/limits（否则被拒，除非有 LimitRange 补默认值）——与 [[concepts/资源限制]] 的 QoS 联动。

三类配额：

| 类型 | 例子 |
| --- | --- |
| **计算** | `requests.cpu`、`limits.cpu`、`requests.memory`、`limits.memory` |
| **存储** | `requests.storage`、`persistentvolumeclaims`、按 storageclass 限额、`*.ephemeral-storage` |
| **对象数** | `pods`、`configmaps`、`secrets`、`services`、`services.loadbalancers`、`services.nodeports` |

**配额范围（scopes / scopeSelector）**：Terminating / NotTerminating / BestEffort / NotBestEffort，或按 **PriorityClass** 分层（高优负载一份配额、普通负载另一份）。

## LimitRange：限/补单个对象

ResourceQuota 管"总量"，LimitRange 管"单个 Pod/容器"：

- 设容器/Pod 的 **min / max**（约束区间）与 **default / defaultRequest**（没写时自动补）。
- 二者常配套：LimitRange 保证每个容器都有合理 requests（从而能进 ResourceQuota 统计），ResourceQuota 守住 namespace 上限。

## requests 与 limits 的区别

计算配额围绕这两者,先分清(详见 [[concepts/资源限制]]):

| | `requests`（请求） | `limits`（上限） |
| --- | --- | --- |
| 含义 | **保底**、预留 | **封顶**、绝不能超 |
| 谁用 | **[[entities/kube-scheduler]] 调度**：节点装得下所有 Pod 的 requests 之和才调上去（**只看 requests**） | 内核 cgroups 运行时强制 |
| 超限 | 装不下就不调度 | **CPU→限速(throttle)**；**内存→OOMKilled**（内存不可压缩，只能杀） |

- `requests ≤ limits`，中间是"能用就用（若节点有空闲）"的弹性区。
- 二者组合决定 **QoS**：`requests==limits`→**Guaranteed**（最后被驱逐）、`requests<limits`→**Burstable**、都不设→**BestEffort**（节点压力下最先杀）。
- 与配额的关系：开了计算配额后**新容器必须有 requests/limits**（否则被拒），这正是 LimitRange 补默认值的意义。

## 配置示例

**ResourceQuota（限 namespace 总量）**：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-quota, namespace: team-a }
spec:
  hard:
    requests.cpu: "10"           # 所有容器 requests.cpu 之和 ≤ 10 核
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    requests.storage: 500Gi      # 所有 PVC 申请容量之和
    persistentvolumeclaims: "5"
    pods: "50"
    services.loadbalancers: "2"
    configmaps: "20"
```

> 语义：该 namespace 内相应资源**总和**不得超过 `hard`，超了新建即被准入控制拒绝。

**LimitRange（限/补单个容器、PVC）**：

```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: team-limits, namespace: team-a }
spec:
  limits:
  - type: Container
    default:        { cpu: 500m, memory: 512Mi }   # 没写 limits 时自动补
    defaultRequest: { cpu: 100m, memory: 128Mi }   # 没写 requests 时自动补
    min:            { cpu: 50m,  memory: 64Mi }    # 单容器下限(越界拒)
    max:            { cpu: "2",  memory: 2Gi }     # 单容器上限(越界拒)
  - type: PersistentVolumeClaim
    min: { storage: 1Gi }
    max: { storage: 100Gi }
```

**两者配合**：开发者提交没写资源的 Pod → **LimitRange 补上默认 requests/limits** → 带着这些值去撞 **ResourceQuota 总量**,没超才准入。缺 LimitRange，Quota 会因"没写 requests"拒掉一堆 Pod；缺 Quota，挡不住"很多小 Pod 累计吃光"。

## 相关

- 单容器资源与 QoS：[[concepts/资源限制]]
- 命名空间隔离：[[entities/Namespace]]
- 配额怎么被准入控制拦下：[[entities/kube-apiserver]]（准入阶段）
