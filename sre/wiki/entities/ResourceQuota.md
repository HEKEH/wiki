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

## 相关

- 单容器资源与 QoS：[[concepts/资源限制]]
- 命名空间隔离：[[entities/Namespace]]
- 配额怎么被准入控制拦下：[[entities/kube-apiserver]]（准入阶段）
