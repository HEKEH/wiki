---
title: Pod
date: 2026-06-23
tags: [kubernetes, pod, 容器, 生命周期, init-container, sidecar]
sources: [kubernetes/kubernetes基本概念.md, kubernetes/kubernetes 101.md, kubernetes/objects/pod.md]
---

# Pod

Pod 是 Kubernetes **调度的基本单位**，是一组紧密关联的[[concepts/容器]]集合，每个 Pod 可包含一个或多个容器。

## 为什么需要 Pod 这一层

Pod 是介于"单个容器"与"整台 [[entities/Node]]"之间、有意设计的**最小调度与共享单位**。即使大多数 Pod 只含一个容器，这层抽象仍解决了直接在 Node 上跑容器无法解决的问题：

- **表达"一组容器是一伙的"**：紧密协作的辅助容器（sidecar，如日志收集、服务网格代理、配置同步）必须与主容器同机、共享网络和文件才能工作。只有"单容器"这一个单位时，无法声明这种归属关系。
- **原子调度**：Pod 内的容器保证被**一起调度到同一 Node、同生共死**，不会被分到不同机器而导致 localhost / 共享卷失效。
- **简化网络模型**：K8s 采用"**每个 Pod 一个 IP**"（而非每容器一个），Pod 内容器共享该 IP，避免端口冲突与 IP 数量爆炸；[[entities/Service]] 面向的也是 Pod IP。
- **统一挂载点**：[[concepts/Volume存储]]、生命周期、[[concepts/健康检查]] 都以 Pod 为单位统一管理，Pod 内多容器可共享同一卷。
- **与运行时解耦**：Pod 是运行时无关的抽象，K8s 只管调度/管理 Pod，容器具体用哪个运行时跑交给 CRI 插件（见 [[concepts/插件机制与可扩展性]]），换运行时不影响核心模型。

> 一句话：直接在 Node 上跑容器，缺的是"一组容器共享网络/存储/命运、并作为整体被调度"的能力，Pod 正是为此而生。

## 关键特性

- **共享命名空间**：Pod 内的多个容器共享 IPC 和 Network namespace，因此可通过进程间通信（IPC）和文件共享高效协作。
- **共享网络与文件系统**：同一 Pod 内的容器共享网络（同一 IP/端口空间）和挂载的 [[concepts/Volume存储]]。每个 Pod 有独立 IP（含同机 Pod），详见 [[concepts/Pod网络模型]]。
- **调度单位**：K8s 调度的是 Pod 整体，而非单个容器——同一 Pod 的容器总在同一 [[entities/Node]] 上运行。
- **生命周期短暂**：Pod 出现异常时会被销毁并由新 Pod 替代；其 IP 会随重启变化，因此不应直接用 Pod IP 交互（用 [[entities/Service]] 代替）。

## 定义方式

所有 K8s 对象都用 manifest（YAML 或 JSON）定义。一个最简 nginx Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

通过 `kubectl create -f nginx.yaml` 创建。

> 生产环境**不直接建裸 Pod**：裸 Pod 一旦调度即与 Node 绑定，**Node 挂掉不会重新调度**（直接删除）。用控制器（[[entities/Deployment]]/[[entities/StatefulSet]]/[[entities/DaemonSet]]/[[entities/Job]]）托管才有容错（见 [[concepts/工作负载控制器]]）。

## 生命周期与重启

- **Phase**（`status.phase`，粗粒度状态）：`Pending`（已创建，镜像还在拉等）→ `Running` → `Succeeded` / `Failed`；`Unknown` 通常是 apiserver 连不上 kubelet。
- **restartPolicy**：`Always`（默认）/ `OnFailure` / `Never`。**只在 Pod 所在 Node 本地重启容器，不会换节点**；换节点是上层控制器重建新 Pod 的事。
- **优雅终止**：删除时 kubelet 先给容器发 `SIGTERM`，等 grace period（默认 30s）再 `SIGKILL`；可用 `preStop` 钩子做清理。

## Init 容器与 sidecar

- **Init 容器**：在应用容器前**按序 run-to-completion**（一个跑完退出才跑下一个），全部成功才启动应用容器；常用于初始化配置、等待依赖、跑迁移。镜像独立，可装应用镜像里不便包含的工具。

- **传统 sidecar 的痛点**：把辅助容器（日志/网格代理如 Envoy）放进 `containers` 与主容器**并列**——但 `containers` 里的容器**并行启动、无启停顺序**：① 代理没就绪主容器就发请求 → 早期请求失败；② 关闭时代理可能**先于主容器死** → 尾部请求丢；③ 在 [[entities/Job]] 里 sidecar 常驻不退 → **Job 永远判不了"完成"**。

- **原生 sidecar** 解决上述问题：把 sidecar 写成 **`initContainers` 里、单独设 `restartPolicy: Always` 的容器**。K8s 对它特殊对待——它常驻不退出，故 K8s 改为**等它"就绪"**（而非退出）就放行下一步。于是它同时获得：**先于主容器启动**（在 init 序列里）+ **贯穿 Pod 生命**（Always 常驻）+ **晚于主容器关闭**，且在 Job 里**不阻止完成**。复用 initContainers 是因其本就自带"排序/先启动"机制。（`v1.28` alpha → `v1.29` beta 默认开 → **`v1.33` GA**）

  ```yaml
  initContainers:
  - { name: setup, image: busybox, command: ["sh","-c","echo prepare"] }  # 普通 init:跑完退出
  - name: proxy
    image: envoy
    restartPolicy: Always     # ← 这一行使它成为"先启动+常驻"的 sidecar
  containers:
  - { name: app, image: myapp }
  # 顺序:setup 跑完 → proxy 就绪 → app 启动;proxy 全程在、且晚于 app 关闭
  ```

| | 普通 Init | 传统 sidecar(`containers`) | 原生 sidecar(init+`Always`) |
| --- | --- | --- | --- |
| 启动 | 主容器前、按序 | 与主容器并行、**无序** | 主容器前(**有序**) |
| 运行多久 | 跑完就退 | 贯穿 Pod 生命 | 贯穿 Pod 生命 |
| 关闭 | (已退出) | **无保证**(可能先死) | **晚于主容器** |
| Job 里 | 正常 | **阻止 Job 完成** | 不阻止 |

## 多容器协作模式

Pod 内多容器共享网络/卷，常见模式：**sidecar**（边车，加日志/监控/代理）、**ambassador**（大使，代理对外连接）、**adapter**（适配器，转换数据/协议格式）。这正是"为什么需要 Pod 这一层"的落地（见上文）。

## 镜像拉取策略（imagePullPolicy）

`IfNotPresent`（默认，本地有就不拉）/ `Always`（每次校验远端）/ `Never`（只用本地）。注意 **`:latest` 标签默认 `Always`**；生产应避免 `:latest`（不可复现）。与节点上的镜像管理/GC 相关，见 [[entities/kubelet]]。

## 私有镜像与身份相关字段

- **`imagePullSecrets`**：引用 `kubernetes.io/dockerconfigjson` 类型的 [[entities/Secret]] 拉私有仓库镜像；也可挂到 [[entities/ServiceAccount]] 上让其名下 Pod 自动继承。
- **`serviceAccountName`**：Pod 以哪个 [[entities/ServiceAccount]] 身份访问 API（默认 `default`，token 经投射卷注入）。
- **主机命名空间开关**：`hostNetwork` / `hostPID` / `hostIPC`（共享宿主对应命名空间，见 [[concepts/Pod网络模型]] 端口冲突）、`hostUsers: false`（启用用户命名空间隔离）——均属安全敏感项，详见 [[entities/SecurityContext]]。

## PodDisruptionBudget（PDB）

声明一组 Pod **同时可用的最小数量**（`minAvailable`/`maxUnavailable`），约束**自愿中断**（如 `kubectl drain` 节点维护、滚动升级）一次别端掉太多副本。作用于 Deployment/ReplicaSet/StatefulSet 管理的 Pod。

> 较新：**原地资源调整（In-Place Pod Resize，v1.33 Beta）** 可不重启容器改 CPU/内存（`kubectl ... --subresource resize`），并与 VPA 集成。

## 相关

- 通常不直接创建 Pod，而是由 [[entities/Deployment]] → ReplicaSet 来管理。
- 用 [[concepts/Label与Selector]] 被 Service / 控制器选中。
- 操作工具见 [[entities/kubectl]]。
