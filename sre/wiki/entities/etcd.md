---
title: etcd
date: 2026-06-23
tags: [kubernetes, etcd, 存储, raft, 一致性]
sources: [kubernetes/kubernetes简介.md, kubernetes/kubernetes集群.md]
---

# etcd

etcd 是一个**分布式、强一致的键值存储（key-value store）**。在 Kubernetes 中，它是**唯一的集群状态数据库**——整个集群的所有对象（[[entities/Pod]]、[[entities/Service]]、[[entities/Deployment]]、[[entities/Node]]、Secret 等）都以 key/value 形式持久化在 etcd 里。

## 命名由来

etcd 并非首字母缩写，而是 **`/etc` + `d`** = "distributed `/etc`"（分布式的 `/etc`）：`/etc` 是 Unix/Linux 存放配置文件的目录，`d` 代表 distributed（也合乎 `sshd`/`httpd` 等守护进程以 `d` 结尾的习惯）。寓意把单机的 `/etc` 配置存储扩展成跨多机的分布式配置/状态存储。由 CoreOS 团队开发，现为 CNCF 项目。

> 源文档原话："etcd 保存了整个集群的状态。"

## 在 Kubernetes 中的角色

- **唯一数据源（single source of truth）**：K8s 是 [[concepts/声明式API]] 系统，"期望状态"与"当前状态"都存在 etcd。其余组件本身不保存状态，全部从 etcd 读写。
- **只有 apiserver 直接访问**：scheduler、controller manager、kubelet 等都不直连 etcd，而是经 apiserver 间接读写。etcd 只有一个"守门人"，便于统一做认证、校验、审计（见 [[concepts/集群架构]]）。
- **属于控制平面**：部署在控制节点侧，是集群最关键、最需要备份保护的组件——etcd 数据丢失等同于整个集群状态丢失。

## 关键特性

- **强一致 + 高可用**：基于 **Raft 共识算法**在多节点间复制数据，通常部署奇数个节点（3 / 5 个）以容忍少数节点故障。
- **Watch 机制**：客户端可监听某个 key 的变化并实时收到通知。这是 K8s 控制器 **reconcile loop** 的基础——控制器通过 apiserver watch etcd，感知期望与现状的偏离后采取动作。

## 相关

- 集群组成与 apiserver 的关系：[[concepts/集群架构]]
- 核心组件全貌：[[entities/Kubernetes]]
