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

端到端完整例子——身份(SA)→ 权限(Role)→ 绑定(RoleBinding)→ Pod 使用,四个对象串起来:

```yaml
# ① ServiceAccount:专用身份(别用 default)
apiVersion: v1
kind: ServiceAccount
metadata: { name: pod-reader-sa, namespace: default }
---
# ② Role:定义"能做什么"——只读 Pod
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: default }
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# ③ RoleBinding:把权限授予这个 SA
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: read-pods, namespace: default }
subjects:
- { kind: ServiceAccount, name: pod-reader-sa, namespace: default }
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
---
# ④ Pod:以这个 SA 身份运行
apiVersion: v1
kind: Pod
metadata: { name: reader, namespace: default }
spec:
  serviceAccountName: pod-reader-sa      # ← 关联 SA(不写则用 default)
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ["sh","-c","kubectl get pods && sleep 3600"]
```

协作:Pod 里的 `kubectl` 凭投射 token 认证成 `pod-reader-sa` → apiserver 认证(你是谁)→ RBAC 查到 `read-pods` 把 `pod-reader` 绑给了它 → 放行 `get pods`;但 `delete pod`、`get secrets` 因没授权被拒。

> ⚠️ **别把权限授给 `default` SA**:它被 namespace 内所有没指定 SA 的 Pod 共用,给它加权等于给一大片 Pod 加权,违反最小权限。**应建专用 SA**(如上 `pod-reader-sa`),仅需要的 Pod 用 `serviceAccountName` 关联。

## token 的现代形态（重要变化）

- ⚠️ **v1.24 起不再自动为 SA 生成长期 token Secret**。Pod 拿到的是经 **TokenRequest** 签发、**有时效、绑定该 Pod** 的 token（投射卷自动轮转）——比旧的永不过期 Secret 安全得多。
- **v1.33（Stable）** 再加固：token 含唯一标识符（审计）、可**绑定到特定节点**、生命周期更可控。

## 相关

- 凭证如何存储：[[entities/Secret]]（`kubernetes.io/service-account-token`）
- 访问控制全链路：[[entities/kube-apiserver]]（认证/授权/准入）
