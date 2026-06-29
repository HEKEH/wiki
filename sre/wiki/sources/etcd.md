---
title: 源文档 - etcd（components）
date: 2026-06-27
tags: [kubernetes, etcd, raft, watch, source]
sources: [kubernetes/components/etcd.md]
---

# 源文档摘要：etcd（components）

深入 etcd 的 Raft 一致性、v2/v3 存储与 Watch、读一致性、对比 Zookeeper/Consul——是 etcd 知识最深的一篇源文档。

## 关键内容

- **Raft**：选举（多数票、Term、150~300ms 随机退避）、日志复制（多数 ACK 后 commit）、安全性（Election Safety、Leader Completeness）、失效处理；wal 二进制日志（type/term/index/data）。
- **v2 vs v3**：v2 纯内存树 + HTTP，Watch 有 1000 条 EventHistory 上限、易丢变更；v3 btree+boltdb、gRPC、多版本（revision）、范围 watch、lease 过期、需 compact（默认 2GB quota）。
- **读一致性**：linearizable（走 Raft，默认）vs serializable（读本地，更快但可能旧）。
- **对比**：etcd vs Zookeeper（能力相近，生态不同）vs Consul（聚焦服务发现+配置）。
- **局限**：Raft 只复制不扩容，扩容需分片（multi-group Raft，如 TiKV/CockroachDB）。

## 勘误 / 时效性

- **v2 已在 K8s v1.11 弃用**（源已注明），现集群用 v3；原理性内容基本无时效问题。

## 关联

详见实体页 [[entities/etcd]] · [[concepts/集群架构]] · [[concepts/控制平面与控制循环]]
