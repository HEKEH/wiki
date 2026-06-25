---
title: Namespace
date: 2026-06-23
tags: [kubernetes, namespace, 资源隔离]
sources: [kubernetes/kubernetes基本概念.md]
---

# Namespace

Namespace 是对一组资源和对象的**抽象集合**，用于将集群内的对象划分为不同的项目组或用户组，实现**逻辑/管理隔离**（注意：默认**不含网络隔离**，详见下文）。

## 关键点

- 常见的 **pods、services、replication controllers、deployments** 等资源都属于某一个 namespace（默认是 `default`）。
- **node、persistentVolumes** 等资源**不属于任何 namespace**（它们是集群级资源）。

> 注意：这里的 Namespace 是 Kubernetes 的资源隔离概念，与 Linux 内核用于[[concepts/容器]]隔离的 namespace 不是同一层东西（虽然命名相同）。

## 隔离了什么、没隔离什么

Namespace 提供的是**管理/逻辑隔离，而非网络隔离**——它是"给资源分组、划管理边界"，不是"防火墙"。

| 维度 | 是否隔离 | 说明 |
| --- | --- | --- |
| 名称作用域 | ✅ | 不同 namespace 可有同名资源（都叫 `nginx` 的 Service 互不冲突） |
| 权限（RBAC） | ✅ 可 | 可授权某人只能操作某个 namespace |
| 资源配额（ResourceQuota） | ✅ 可 | 可限制每个 namespace 的 CPU/内存用量 |
| `kubectl` 默认范围 | ✅ | `kubectl get pods` 默认只看当前 namespace |
| **网络访问** | ❌ **默认不隔离** | 跨 namespace 的 Pod/[[entities/Service]] **默认可互相访问** |

### 跨 namespace 访问

默认互通，通过 Service 的 DNS 名访问（完整形式 `<service>.<namespace>.svc.cluster.local`）：

```bash
curl http://api.team-b                    # 简写：访问 team-b 里的 api 服务
curl http://api.team-b.svc.cluster.local  # 全称
# 同 namespace 内可省略 namespace：curl http://api
```

### 要真正隔离网络

需额外定义 **NetworkPolicy（网络策略）**——这才是 K8s 里做网络隔离的对象（如"`team-b` 只接受本 namespace 流量"）。但其是否生效**取决于 CNI 插件支持**（Calico/Cilium 支持；不支持的 CNI 即使写了也不生效），见 [[concepts/插件机制与可扩展性]]。

## 相关

- [[entities/Node]]、PersistentVolume 等集群级资源不归属 namespace。
- 配合 [[concepts/Label与Selector]] 可在 namespace 内进一步组织资源。
