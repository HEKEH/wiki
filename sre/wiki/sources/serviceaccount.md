---
title: ServiceAccount（资源对象）
date: 2026-06-30
tags: [kubernetes, serviceaccount, rbac, token, 认证]
sources: [kubernetes/objects/serviceaccount.md]
---

# 源摘要：ServiceAccount

feisky handbook `/concepts/objects/serviceaccount.md` 的要点（深入页见 [[entities/ServiceAccount]]）。

## 核心

- 为 **Pod 内进程调用 K8s API** 设计的身份（对比 User account 为人设计）。SA **局限于所在 namespace**，每个 namespace 自动建一个 `default` SA。
- ServiceAccount 准入控制：Pod 未指定则自动设 `spec.serviceAccountName=default`、校验 SA 存在、把 SA 的 imagePullSecrets 加进 Pod、把 token+ca.crt 挂到 `/var/run/secrets/kubernetes.io/serviceaccount/`。
- **认证而非授权**：SA 只解决"你是谁"，"能做什么"靠 **RBAC**（Role/ClusterRole + RoleBinding/ClusterRoleBinding）。

## v1.33 绑定令牌安全改进（Stable）

- 唯一令牌标识符（审计）、**节点绑定**令牌、更精细的生命周期（TokenRequest 的 `boundObjectRef`/`expirationSeconds`）。

## 勘误 / staleness

- ⚠️ **v1.24 起不再自动为 SA 创建长期 token Secret**（`LegacyServiceAccountTokenNoAutoGeneration` 默认开）；Pod 改用 **TokenRequest 投射卷**注入短时效、绑定 Pod 的 token。
- ⚠️ RBAC 示例用 `rbac.authorization.k8s.io/v1alpha1`——正式版 **v1（GA 自 v1.8）**；`--authorization-rbac-super-user` 选项已移除。
