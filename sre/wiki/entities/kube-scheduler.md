---
title: kube-scheduler
date: 2026-06-27
tags: [kubernetes, scheduler, 调度, 亲和性, taint, 调度框架]
sources: [kubernetes/components/scheduler.md]
---

# kube-scheduler

kube-scheduler 是控制平面中**专管"放置（placement）"的控制器**：它 watch [[entities/kube-apiserver]]，找出还没分配 [[entities/Node]] 的 [[entities/Pod]]，按调度策略选一台 Node，写回 Pod 的 `spec.nodeName`（绑定）。它本身**不启动容器**——绑定后由目标 Node 的 [[entities/kubelet]] 真正拉起（见 [[concepts/控制平面与控制循环]]）。

调度需权衡：公平、资源利用率、QoS、亲和/反亲和、数据本地化、负载干扰、deadline 等。

## 调度两阶段（经典模型）

```text
所有节点 ──预选(Predicate/Filter)──► 可行节点 ──优选(Priority/Score)──► 选分最高者
```

- **预选（predicate / filter）**：过滤掉不满足硬条件的节点。代表：`PodFitsResources`（CPU/内存/Pod 数够不够）、`PodFitsHostPorts`（端口冲突）、`MatchNodeSelector`、`PodToleratesNodeTaints`、`CheckNodeMemory/DiskPressure`、各类 Volume 数量/Zone 冲突检查。
- **优选（priority / score）**：给可行节点打分选最优。代表：`LeastRequestedPriority`（挑空闲多的）、`BalancedResourceAllocation`（均衡 CPU/内存）、`SelectorSpreadPriority`（同 Service/RS 的 Pod 分散到不同节点）、`InterPodAffinityPriority`、`NodeAffinityPriority`、`ImageLocalityPriority`（镜像已在本地的优先）。

> 这回答了 home.md 的开放问题"调度器预选/优选细节"。注意 predicate/priority 是**经典调度器**词汇；现代实现见下文调度框架（插件化的 Filter/Score 扩展点）。

## 指定/约束调度位置

scheduler 默认自动挑节点；但有些落点要求**它自己推断不出来**，需要你(用户)在 Pod 上写规则**主动干预**。常见动机：① **挑硬件**(要 GPU/SSD)；② **靠近**依赖的 Pod(低延迟，如 web 贴着 Redis)；③ **打散**副本(别全堆一台，抗单点)；④ **隔离/合规**。这些约束会**喂进上一节的预选/优选**——硬要求当预选过滤、软偏好当优选加分。

### 谁挑谁

| 机制 | 方向 | 依据 | 例子 |
| --- | --- | --- | --- |
| **nodeSelector** | Pod 挑节点 | **节点 label**(精确匹配,最简单) | 只去 `disktype=ssd` 的节点 |
| **nodeAffinity** | Pod 挑节点 | 节点 label,更强(操作符 In/NotIn/Exists、硬/软) | 必须在 zone-a/b；**优先**带 GPU 的 |
| **podAffinity / AntiAffinity** | Pod 挑节点 | **节点上已有哪些 Pod** | web 调到"已有 redis 的节点"；副本"不在同一节点" |
| **Taint/Toleration**(下一节) | **节点挑 Pod** | 节点主动排斥,Pod 需容忍 | 节点打 taint,仅容忍者能上 |

> 直觉：前三种是 **Pod 主动"我想去哪"**；Taint 是 **节点主动"我不收谁"**(一拉一推)。label/selector 机制见 [[concepts/Label与Selector]]。

### 硬 vs 软

- **required…(硬)**：**必须**满足，否则不调度 → 进**预选**当过滤条件。
- **preferred…(软)**：**尽量**满足，不满足也能调 → 进**优选**当加分项。

### topologyKey（podAffinity 专用）：约束生效的"范围"

取节点的拓扑标签，决定亲和/反亲和在多大范围内算："同一个"：

- `kubernetes.io/hostname` → 范围是**单台节点**("不在同一节点")。
- `topology.kubernetes.io/zone` → 范围是**可用区**("不在同一可用区"，容灾更强)。

### 最常见两个用法

- **打散副本做 HA**：给 Deployment 的 Pod 加 `podAntiAffinity`(topologyKey=hostname/zone)，副本分散，单点故障不团灭。
- **挑硬件**：`nodeSelector: disktype=ssd` 或 `nodeAffinity` 选 GPU 节点。

## Taints 和 Tolerations（污点与容忍）

**Taint 打在 Node 上排斥 Pod；Toleration 打在 Pod 上表示能容忍**——只有容忍了 Node 全部 taint 的 Pod 才能调度上去。三种 effect：

> **为何"节点挑 Pod"却把 toleration 写在 Pod 里**：taint(节点)和 toleration(Pod)是配对的两半——**节点用 taint 主动设门槛排斥**(这才是"节点挑 Pod"，方向指谁设门槛)；toleration 是 **Pod 自带的"通行证"** 去闯这道门槛，写在 Pod 是因为许可须跟着每个 Pod 走。类比：包厢挂"谢绝入内"(门是节点的)，你得自己揣会员卡(卡在你身上)。
> **关键**：toleration **只放行、不吸引**——它让 Pod *能*上被 taint 的节点，但不保证去。要"定向投放"(如专属 GPU 节点)需**两件一起**：节点 `taint` 挡走闲杂 + Pod `toleration` 放行 + `nodeSelector/nodeAffinity` 把 Pod **拉过去**。

- `NoSchedule`：新 Pod 不调度上来，不影响已运行的。
- `PreferNoSchedule`：软版，尽量不调度。
- `NoExecute`：新 Pod 不上来，**且驱逐已运行的**（Pod 可设 `tolerationSeconds` 延迟驱逐）。

> 这是上面 controller-manager **Node 驱逐**的底层机制：节点 NotReady 时 K8s 给 Node 打 `not-ready`/`unreachable` 的 NoExecute taint，Pod 默认容忍 300s 后被驱逐。

## 优先级与抢占（Pod Priority，v1.11+ 默认开启）

先定义 `PriorityClass`（集群级、非 namespace 资源）设定 value（越大越高），Pod 经 `priorityClassName` 引用。高优先级 Pod 可**抢占**低优先级 Pod 的资源。

## 调度框架（Scheduling Framework，v1.19+）

现代扩展方式：把调度过程拆成一系列**扩展点**（PreFilter/Filter/Score/Reserve/Permit/Bind…），以**插件**形式增删，经 `KubeSchedulerConfiguration` 配置 profile。

> ⚠️ 旧的 `--policy-config-file` 调度策略文件**仅 v1.23 之前支持**，之后须改用调度框架插件。

## 较新特性

- **DRA（Dynamic Resource Allocation）**：v1.26 起调度器原生支持设备资源（如 GPU）动态分配，v1.33 增强设备感知调度、设备 taint/容忍、分区设备、优先级列表。Pod 经 `resourceClaims` 声明设备需求。
- **存储容量评分（Storage Capacity Scoring，v1.33 Alpha）**：扩展 VolumeBinding 插件，按节点可用存储容量给节点打分，取代旧的 `VolumeCapacityPriority`。

## 多调度器

可同时运行多个调度器实例，Pod 经 `spec.schedulerName` 选用（默认内置 `default-scheduler`）。

## 其他影响调度的因素

- Node 处于 `MemoryPressure` → 不调度 BestEffort（无 requests/limits、QoS 最低、最先被杀；QoS 三档见 [[concepts/资源限制]]）的新 Pod。
- Node 处于 `DiskPressure` → 不调度任何新 Pod。
- Critical Pods（`system-cluster-critical` / `system-node-critical`）异常时自动重调度。

## 相关

- 与 controller 的"接力"：[[concepts/控制平面与控制循环]]
- 节点失联驱逐的速率控制：[[entities/kube-controller-manager]]
- 资源 requests/limits 与 QoS：[[concepts/资源限制]]
