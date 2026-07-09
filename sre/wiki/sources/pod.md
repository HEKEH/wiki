---
title: Pod（资源对象）
date: 2026-06-30
tags: [kubernetes, pod, 生命周期, init-container, sidecar]
sources: [kubernetes/objects/pod.md]
---

# 源摘要：Pod

feisky handbook `/concepts/objects/pod.md` 的要点（深入页见 [[entities/Pod]]）。

## 核心

- Pod = 共享 IPC/Network namespace 的一组容器，可 localhost 互通、共享 Volume；K8s 调度的基本单位。
- **无容错性**：直接创建的裸 Pod 一旦调度即与 Node 绑定，Node 挂掉也**不会重新调度**（直接删除）——故推荐用控制器（Deployment/DaemonSet/StatefulSet/Job）托管。
- **优雅终止**：删除时先发 `SIGTERM`，过 grace period 再 `SIGKILL`。

## 关键字段/机制

- **生命周期 Phase**：Pending / Running / Succeeded / Failed / Unknown。
- **restartPolicy**：`Always`（默认）/ `OnFailure` / `Never`；**仅本地重启，不跨 Node**。
- **imagePullPolicy**：`IfNotPresent`（默认）/ `Always` / `Never`；`:latest` 标签默认 `Always`。
- **Init 容器**：run-to-completion、按序执行；v1.29+ 设 `restartPolicy: Always` 即**原生 sidecar**（v1.33 GA）。
- **多容器模式**：sidecar / ambassador / adapter / 配置助手。
- **生命周期钩子**：`postStart` / `preStop`（exec/httpGet/sleep）。
- **PodDisruptionBudget**：保障一组 Pod 同时可用的最小数量。
- **优先级抢占**：`PriorityClass` + `priorityClassName`（见 [[entities/kube-scheduler]]）。
- 调度落点：nodeSelector / nodeAffinity / podAffinity / taint+toleration / nodeName。

## 较新特性（v1.33）

- **原地资源调整（In-Place Pod Resize）** 升 Beta：`kubectl ... --subresource resize` 改 CPU/内存不重启；可与 VPA 集成。
- **用户命名空间隔离**：`hostUsers: false` 把容器内 UID 映射到主机非特权用户。
- `lifecycle.stopSignal` 自定义终止信号（Alpha）；`preStop` 支持零秒 sleep（Beta）。

## 勘误 / staleness

- ⚠️ 源称"仅支持 Docker 镜像"——已过时：经 **CRI** 支持任意 OCI 运行时（containerd/CRI-O），dockershim 于 v1.24 移除。
- ⚠️ `PriorityClass` 示例用 `scheduling.k8s.io/v1alpha1`——现为 **scheduling.k8s.io/v1（GA）**；PodPriority v1.11 起默认开启。
- ⚠️ DNS/示例中的 `kube-dns`、`CustomPodDNS` feature-gate 均已过时——DNS 默认 **CoreDNS**，自定义 DNS（dnsConfig）早已 GA（见 [[entities/CoreDNS]]）。
- `kubectl get --show-all` 标志已移除（已结束的 Pod 默认显示）。
