---
title: SecurityContext
date: 2026-06-30
tags: [kubernetes, securitycontext, psa, selinux, capabilities, 用户命名空间]
sources: [kubernetes/objects/security-context.md]
---

# SecurityContext

SecurityContext 限制容器的权限与行为，保护宿主机与同节点其他容器——是 Pod/容器层面的**最小权限**配置。

## 两个层级

| 层级 | 作用范围 | 常用字段 |
| --- | --- | --- |
| **容器级**（`containers[].securityContext`） | 仅该容器（不含卷） | `privileged`、`capabilities`（add/drop）、`allowPrivilegeEscalation`、`readOnlyRootFilesystem`、`runAsUser/runAsGroup` |
| **Pod 级**（`spec.securityContext`） | 所有容器 + 卷 | `fsGroup`、`supplementalGroups`、`seLinuxOptions` |

加固惯用法：`runAsNonRoot` + `allowPrivilegeEscalation: false` + `readOnlyRootFilesystem: true` + `capabilities.drop: [ALL]`。

## supplementalGroupsPolicy（v1.33 Beta）

控制补充组怎么算：**Merge**（默认，合并容器镜像 `/etc/group` 里的隐式 GID）/ **Strict**（只用清单里显式声明的组，忽略镜像隐式 GID——更安全、可审计）。需 containerd 2.0+ / CRI-O 1.31+。

## 用户命名空间（v1.33 默认启用）

`hostUsers: false`：把容器内 UID（含 root=0）映射到**主机上的非特权用户**——即使容器内拿到 root 也危害不到宿主，显著降低容器逃逸风险。需 Linux 6.3+（推荐）、支持 idmap 的运行时；NFS 卷暂不支持。

## SELinux / Capabilities

- `seLinuxOptions` 给进程与卷打 SELinux 标签（强制访问控制）。
- Linux **capabilities** 把 root 细分为可单独增删的能力位（如只加 `NET_ADMIN`、丢弃其余）。

## 集群级策略：PSP 已被 PSA 取代

> ⚠️ **PodSecurityPolicy（PSP）于 v1.21 弃用、v1.25 移除**。现用内置 **Pod Security Admission（PSA）**：在 namespace 上打 label 选三档标准——**Privileged / Baseline / Restricted**，由准入控制强制（见 [[entities/kube-apiserver]]）。更细的策略用 Kyverno / OPA-Gatekeeper。源文档的 PSP 章节仅历史参考。

## 相关

- Pod 内 hostNetwork/hostPID/hostIPC 等主机命名空间开关：[[entities/Pod]]
- 准入控制如何强制策略：[[entities/kube-apiserver]]
