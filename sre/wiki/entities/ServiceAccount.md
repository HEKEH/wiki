---
title: ServiceAccount
date: 2026-06-30
tags: [kubernetes, serviceaccount, rbac, token, 认证, 身份]
sources: [kubernetes/objects/serviceaccount.md]
---

# ServiceAccount

ServiceAccount（SA）是给 **Pod 内进程**调用 [[entities/kube-apiserver]] 用的身份（机器身份），区别于给"人"用的 User account。

| | User account | ServiceAccount |
| --- | --- | --- |
| 面向 | 人 | Pod 内进程 |
| 作用域 | 跨 namespace | **局限所在 namespace** |
| 来源 | 外部 IdP / 证书 | K8s 内置对象，每 namespace 自动有一个 `default` |

## 自动注入

开了 ServiceAccount 准入控制后，Pod 不指定就自动用 `default` SA，并把 **token + ca.crt + namespace** 挂到 `/var/run/secrets/kubernetes.io/serviceaccount/`；SA 上配的 `imagePullSecrets` 也会自动加进 Pod（省去每个 Pod 重复写，见 [[entities/Secret]] 私有镜像）。

## 认证 ≠ 授权：配合 RBAC

SA 只回答"**你是谁**"（认证）；"**能做什么**"由 **[[concepts/RBAC]]** 决定——`Role`/`ClusterRole` 定义权限，`RoleBinding`/`ClusterRoleBinding` 把权限绑到 SA。这正是 [[entities/kube-apiserver]] 访问控制三关卡里的认证→授权（见该页）。

```yaml
roleRef: { kind: Role, name: pod-reader }
subjects: [{ kind: ServiceAccount, name: default }]
```

## token 的现代形态（重要变化）

- ⚠️ **v1.24 起不再自动为 SA 生成长期 token Secret**。Pod 拿到的是经 **TokenRequest** 签发、**有时效、绑定该 Pod** 的 token（投射卷自动轮转）——比旧的永不过期 Secret 安全得多。
- **v1.33（Stable）** 再加固：token 含唯一标识符（审计）、可**绑定到特定节点**、生命周期更可控。

## 相关

- 凭证如何存储：[[entities/Secret]]（`kubernetes.io/service-account-token`）
- 访问控制全链路：[[entities/kube-apiserver]]（认证/授权/准入）
