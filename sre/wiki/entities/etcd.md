---
title: etcd
date: 2026-06-23
tags: [kubernetes, etcd, 存储, raft, 一致性]
sources: [kubernetes/kubernetes简介.md, kubernetes/kubernetes集群.md, kubernetes/components/etcd.md]
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

## Raft 共识（一致性怎么来的）

etcd 用 **Raft** 在多节点间保持强一致，核心是"选主 + 日志复制"：

- **选举**：节点初始为 follower，带一个 election timeout；超时未收到 leader 心跳就转 candidate 发起投票，**获多数票（n/2+1）即成 leader**。冲突时各 candidate 随机退避 150~300ms 再投，避免反复平票。每选一次，任期 **Term +1**。
- **日志复制**：leader 收到写请求先 append 到本地 log，经心跳同步给 follower；收到**多数** follower 的 ACK 后才把该 entry 标记为 **committed** 并落盘，再通知客户端与其余 follower。
- **安全性**：每个 Term 至多一个 leader（Election Safety）；已 committed 的日志必出现在后续所有 leader 上（Leader Completeness）——选举时用 Term/Index 比较保证新 leader 包含全部已提交日志。
- **失效处理**：leader 挂 → 其他节点超时重新选举，旧 leader 恢复后变 follower（其未提交日志被覆盖）；follower 挂 → 恢复后从 leader 重新复制即可。

> 这也是为何 etcd **需要奇数个节点且依赖"多数派（quorum）"**——少数节点故障仍能选主和提交，网络分区中少数派一侧无法写入。Raft 选 etcd 集群自己的 leader，与 [[entities/kube-controller-manager]] 用 Lease 锁在多副本间选主是**两个层面**的选主。

### Raft 日志（WAL）里有什么

Raft 日志记录的是**一串有序的「变更操作」，而非当前值**——各节点回放同一串日志即得到一致状态（当前 KV 是日志 apply 到后端 boltdb 的结果）。每条日志条目（LogEntry）含 4 字段：

| 字段 | 含义 |
| --- | --- |
| **type** | `Normal`（普通数据操作）/ `ConfChange`（集群成员变更，如增删节点） |
| **term** | 产生该条目的 leader 任期 |
| **index** | 严格递增的变更序号，leader/follower 据此对齐日志 |
| **data** | protobuf 二进制，那次 put/delete/txn 请求的实际内容 |

- 只有**写/变更**进日志，**读不写日志**（线性一致读用 ReadIndex 确认 leader）。
- 日志先顺序写 **WAL** 落盘（崩溃可重放）；定期打 **snapshot** 后旧 WAL 可丢弃，避免无限增长（与下文 revision compact / 2GB quota 配套）。

## v2 与 v3

v2、v3 共享同一套 Raft 代码，但**接口/存储/数据相互隔离**（v2 数据只能用 v2 接口访问）。K8s 推荐用 **v3**，**v2 已在 K8s v1.11 弃用**。

| | etcd v2 | etcd v3 |
| --- | --- | --- |
| 接口 | HTTP/JSON | **gRPC**（长连接，效率高） |
| 存储 | 纯内存树结构，定期序列化为 JSON 落盘 | 内存 **btree 索引** + 后端 **boltdb**（支持事务） |
| 数据模型 | 目录树 | 纯 KV（前缀模拟目录），**多版本**（key=revision） |
| Watch | 只能 watch 单 key/子树，**EventHistory 上限 1000 条**，断连可能丢变更 | watch 单 key 或**范围**，支持**从任意版本开始**，基本可实现完整同步 |
| 过期 | TTL 设在每个 key 上 | TTL 设在 **lease** 上，多 key 关联同一 lease 统一过期/批量续约 |

> **revision**：v3 每次事务 main rev +1、事务内每步 sub rev +1，boltdb 按 revision 保存每个历史版本；故需 **compact** 回收旧版本，否则触达默认 **2GB backend quota** 后报 `database space exceeded` 而无法写入。

## Watch 与读一致性

- **Watch**：v3 的 WatchableStore 分 synced/unsynced 两组 watcher，后台 goroutine 持续把落后的 unsynced 追平再迁入 synced；推送阻塞时进重试队列而非直接断连（修复了 v2 的痛点）。这正是 K8s 控制器 reconcile loop 的事件来源。
- **读一致性**：v3 默认 `--consistency="l"`（linearizable）走 Raft 读，保证一致但有性能损耗、网络分区时少数派不能提供一致读；可选 serializable 读本地数据（更快但可能读到旧值）。

## 对比 Zookeeper / Consul

- **etcd vs Zookeeper**：能力相近（通用一致性元信息存储 + watch）。Zookeeper（Apache/Java，从 Hadoop 孵化，生态老：Hadoop/Kafka/Mesos…）；etcd（CoreOS，REST/gRPC、易用、社区活跃，被 K8s 采用）。
- **etcd/Zookeeper vs Consul**：前两者提供**通用一致性存储**（服务发现等需自行实现）；Consul 直接以**服务发现 + 配置变更**为目标，附带 KV。

> **容量局限**：Raft 只解决数据**复制/同步**，加节点不增容量——扩容需分片（multi-group Raft，如 TiKV/CockroachDB），etcd 本身不做。

## 相关

- 集群组成与 apiserver 的关系：[[concepts/集群架构]]
- 控制器选主（另一层面）：[[entities/kube-controller-manager]]
- 核心组件全貌：[[entities/Kubernetes]]
