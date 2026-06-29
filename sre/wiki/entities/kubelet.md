---
title: kubelet
date: 2026-06-26
tags: [kubernetes, kubelet, 节点, agent]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes架构原理.md, kubernetes/components/kubelet.md]
---

# kubelet

kubelet 是每个 [[entities/Node]] 上的**主 agent**（每个 Node 一个）。总职责：**接收"本节点该跑哪些 [[entities/Pod]]"，确保这些 Pod 的容器按期望运行，并把现状上报回 apiserver**（见 [[concepts/控制平面与控制循环]] 端到端链路的第 ④⑤ 步）。

## 与分层架构的关系

kubelet **不等于核心层**。[[concepts/分层架构]] 的"核心层"是按功能高度划分的**概念层**，kubelet 是**具体组件**——它实现了核心层"对内提供插件式应用执行环境"的那一块，但核心层更大（还含对外的核心 API）。即：**kubelet 承担核心层的执行环境角色，但核心层 ≠ kubelet**。

## 内部模块 / 职责

| 模块 | 作用 |
| --- | --- |
| **syncLoop（主循环）** | 心脏：不断比对"期望的 Pod"与"实际运行的容器"，驱动收敛 |
| **CRI 客户端** | 调用 container runtime 拉镜像、创建沙箱、起停容器（见 [[concepts/插件机制与可扩展性]]） |
| **PLEG**（Pod Lifecycle Event Generator） | 监听运行时容器状态变化并通知主循环 |
| **Volume Manager** | 挂载/卸载卷，对接 CSI（见 [[concepts/Volume存储]]） |
| **网络（CNI）** | 起 Pod 时配置网络、分配 IP（见 [[concepts/Pod网络模型]]） |
| **Probe Manager** | 执行 liveness/readiness/startup 探针（见 [[concepts/健康检查]]） |
| **Image Manager** | 镜像拉取与垃圾回收 |
| **cAdvisor** | 采集容器 CPU/内存等资源指标 |
| **Container/cgroup 管理** | 按 [[concepts/资源限制]] 通过 cgroups 约束资源、管理 QoS |
| **Eviction Manager** | 节点资源紧张时驱逐 Pod |
| **Device Manager** | 接入 GPU 等设备插件 |
| **Node 状态/心跳** | 注册节点、定期上报 NodeStatus / node lease |

## 作为接口的调用方

kubelet 是 **CRI / CNI / CSI 的客户端（调用方）**——它调用这些接口让容器跑起来、联网、挂卷；具体实现（containerd、Calico、CSI driver）由生态系统层的插件提供（见 [[concepts/插件机制与可扩展性]]）。

> 注意：kubelet 与 [[entities/kubectl]] 是完全不同的东西——kubelet 是节点 agent，kubectl 是命令行工具。

## 静态 Pod 与自举

**manifest** 指描述 K8s 对象期望状态的 YAML/JSON 文件（即 [[concepts/声明式API]] 的输入）。普通 manifest 由人/工具写好、经 `kubectl apply` 交给 apiserver 生效；而 kubelet 还能**直接管理"静态 Pod（static Pod）"**——读取**节点本地目录**（由 kubelet 配置 `staticPodPath` 指定，kubeadm 默认 `/etc/kubernetes/manifests/`）里的 manifest 起 Pod，**完全不经过 apiserver、不需要 scheduler**。

| | 普通 manifest | 本地/静态 manifest |
| --- | --- | --- |
| 来源 | 人 / Helm / Kustomize 编写 | 引导工具（kubeadm）**生成** |
| 位置 | 你的仓库 → 提交集群 | 节点 `/etc/kubernetes/manifests/` |
| 生效 | `kubectl apply` → apiserver | kubelet **直接读目录** |

这是**自举控制平面**的方式：`kubeadm init` 自动生成 apiserver / controller-manager / scheduler / [[entities/etcd]] 的静态 Pod manifest 写入该目录，kubelet 一启动就把它们跑起来——解决"apiserver 还没起、谁来起 apiserver"的循环依赖（见 [[concepts/设计理念]] 引导原则）。

> 静态 Pod 起来后，kubelet 会在 apiserver 里建一个**只读的"镜像 Pod（mirror pod）"**，所以 `kubectl get pods` 也能看到它们，但它们由 kubelet 管、非 scheduler 调度。

## Pod 清单来源与创建细节

kubelet 以 **PodSpec** 工作，从四种来源接收"本节点该跑哪些 Pod"：

- **API Server**（主，常规路径）：watch+list `/registry/pods` 中绑定到本节点的 Pod。
- **本地文件**（static Pod）：默认 `/etc/kubernetes/manifests/`，定期重扫。
- **HTTP endpoint**（`--manifest-url`，**拉**）：kubelet 定时 GET 该 URL 取清单。
- **HTTP server**（**推**）：kubelet 起监听口，等外部把清单 POST 过来。

> file / http-url / http-server 三者产生的都是 **static Pod**（绕过 apiserver/scheduler，kubelet 直管、建 mirror pod 可见）；**只有 API Server 是常规路径**。后两种 HTTP 方式实践中几乎不用，现实里 static Pod 用本地目录、其余走 apiserver。

创建 Pod 时关键一步：先用 **pause（sandbox）容器**为整个 Pod 建好网络命名空间，其他容器再共享它的网络——这正是 [[entities/Pod]] "一组容器共享网络"的实现根基（见 [[concepts/Pod网络模型]]）。随后挂卷、拉 Secret、按 CRI 拉镜像起容器。

## CRI：客户端 vs 服务端

kubelet 通过 **CRI（Container Runtime Interface，v1.5 引入）** 与容器运行时解耦：CRI 基于 gRPC 定义 `RuntimeService` 与 `ImageService`。**kubelet 是 CRI 客户端，容器运行时实现服务端（CRI shim，监听本地 Unix socket）**。实现有 containerd、CRI-O 等，底层 OCI 引擎有 runc、gVisor、Kata 等。

> ⚠️ 源文档以 Docker/dockershim 为默认运行时——**dockershim 已在 v1.24 移除**，Docker 不再是内置默认；`--network-plugin` 选项亦已移除（CNI 成为唯一方式）；Heapster 早已废弃，节点/容器指标改由 metrics-server 经 `/metrics/resource` 提供，**cAdvisor 4194 端口也已移除**。

## 驱逐信号（Eviction）

Eviction Manager 监控资源并在触达阈值时停 Pod、置 `PodPhase=Failed`：

| 信号（节点剩余资源） | 跌破阈值触发的 Condition |
| --- | --- |
| `memory.available`（可用内存） | MemoryPressure |
| `nodefs.available` / `nodefs.inodesFree` | DiskPressure |
| `imagefs.available` / `imagefs.inodesFree` | DiskPressure |

读表要点：

- **信号**=节点某资源剩余量；**Condition**=越线后给 Node 打的状态标记。阈值用 `--eviction-hard=memory.available<500Mi,...` 配。
- **两个文件系统**：`nodefs`=节点主盘(Pod 卷、容器日志)；`imagefs`=放镜像+容器可写层的盘(可能独立,也可能同 nodefs)。
- **available vs inodesFree**：前者是剩余**空间(字节)**，后者是剩余 **inode(文件数)**——小文件太多会先耗尽 inode,即便有空间也触发 DiskPressure。
- 越线后：打 Condition → 先**回收**(删停止的 Pod、清未用镜像)→ 不够再按 QoS 驱逐 → 该 Condition 还反馈给 [[entities/kube-scheduler]](MemoryPressure 拒 BestEffort 新 Pod、DiskPressure 拒一切新 Pod)。

- **软驱逐**：达阈值并超过宽限期才动手；**硬驱逐**：达阈值立即驱逐。
- 驱逐用户 Pod 的顺序：**BestEffort → Burstable → Guaranteed**（QoS 越低越先被驱，见 [[concepts/资源限制]]）。
- 与 [[entities/kube-controller-manager]] 的"Node 驱逐"区分：这是 **节点本地**因资源紧张主动驱逐；后者是 **控制面**因节点失联远程驱逐。

## 相关

- 所在节点与同级组件：[[entities/Node]]（kubelet / container runtime / kube-proxy）
- 控制平面协作：[[concepts/控制平面与控制循环]]
