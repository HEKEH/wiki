---
title: DaemonSet（资源对象）
date: 2026-06-30
tags: [kubernetes, daemonset, 节点守护, 日志, 监控]
sources: [kubernetes/objects/daemonset.md]
---

# 源摘要：DaemonSet

feisky handbook `/concepts/objects/daemonset.md` 的要点（深入页见 [[entities/DaemonSet]]）。

## 核心

- 保证**每个 Node 上运行一个 Pod 副本**；典型用途：日志收集（fluentd）、监控（node-exporter）、系统程序（kube-proxy、CNI、ceph/glusterd）。
- **忽略 Node 的 unschedulable 状态**（仍会在被标记不可调度的节点上跑）。

## 限定运行节点

- `nodeSelector`（label 精确匹配）/ `nodeAffinity`（更丰富）/ `podAffinity`。

## 更新与回滚

- `.spec.updateStrategy.type`：**RollingUpdate** / **OnDelete**；RollingUpdate 可设 `maxUnavailable`、`minReadySeconds`。
- 支持 `kubectl rollout history/undo/status`（v1.7+）。

## 与静态 Pod 对比

- 也可用 **静态 Pod**（kubelet `--pod-manifest-path` 目录）在每台机器跑指定 Pod；静态 Pod 不能经 API Server 删除，只能删 manifest 文件（见 [[entities/kubelet]]）。

## 勘误 / staleness

- ⚠️ 源称"OnDelete 是默认更新策略"——已过时：**apps/v1 默认是 RollingUpdate**。
- ⚠️ 示例容忍 `node-role.kubernetes.io/master`——新版控制节点 taint 为 `node-role.kubernetes.io/control-plane`（见 [[entities/kubeadm]]）。
