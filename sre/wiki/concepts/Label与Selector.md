---
title: Label 与 Selector
date: 2026-06-23
tags: [kubernetes, label, selector, annotation]
sources: [kubernetes/kubernetes基本概念.md]
---

# Label 与 Selector

Label（标签）是识别 Kubernetes 对象的 **key/value** 标记，附加到对象上；Label Selector 则用来按 label 选择一组对象。这是 K8s 中**松耦合关联资源**的核心机制。

## Label

- 以 key/value 形式附加到对象（key 最长 63 字节；value 可为空或不超过 253 字节）。
- **不提供唯一性**——很多对象（如一组 Pod）常用相同 label 标志同一应用。

label 写在对象的 `metadata.labels` 下，key 名由用户自定义（`env`、`app` 等都不是内置的）：

```yaml
metadata:
  labels:
    app: nginx
    env: production   # selector 中的 env 即指这里定义的 key
```

## Label Selector

定义好 label 后，其他对象用 Selector 选中一组相同 label 的对象（如 ReplicaSet、[[entities/Service]] 用它选一组 [[entities/Pod]]）。支持：

- **等式**：`app=nginx`、`env!=production`
- **集合**：`env in (production, qa)`
- **多 label（AND 关系）**：`app=nginx,env=test`

## selector 重叠的坑

不同 [[entities/Deployment]]（或其他控制器）的 `matchLabels` **必须互不重叠**——K8s 不做唯一性校验，避免重叠是使用者的责任。

- **重叠后果**：不是"一个 Pod 同属两者"（一个 Pod 最多一个控制 owner），而是**控制器互相抢/误删/错误计数副本**，行为不可预测。
- **部分安全网 `pod-template-hash`**：Deployment 会给所属 ReplicaSet 自动加一个按 Pod 模板算出的哈希 label，使**不同 Deployment 的 RS 不会误抢对方的 Pod**；但它**挡不住两个 Deployment 共用同一 `spec.selector`**（你写的 selector 不含该哈希）。
- **最佳实践**：每个 Deployment 用**唯一的应用标识**（如 `app: frontend`）做 selector，别用 `env`、`tier` 这种多应用共享的宽泛 label 单独做 selector。selector 创建后不可改，需一开始设计好。

## Annotations（对比）

Annotations 同样是 key/value 附加于对象，但**用途不同**：

- Label/Selector 用于**标志和选择**对象。
- Annotations 用于**记录附加信息**（辅助部署、安全策略、调度策略等），不用于选择。例如 Deployment 用 annotations 记录 rolling update 状态。

annotations 写在 `metadata.annotations` 下，常存较长或结构化的信息（无 value 长度限制，不能被 selector 选择）：

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "update image to nginx:1.9.1"  # 记录变更原因
    description: "前端 nginx 服务，由 web 团队维护"
```

## 意义

Label/Selector 是 [[entities/Service]] 找到后端 Pod、[[entities/Deployment]] 管理副本的基础，体现了 K8s 用"声明式标记 + 选择"而非硬编码引用来组织资源（见 [[concepts/声明式API]]）。
