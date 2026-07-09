---
title: SecurityContext（资源对象）
date: 2026-06-30
tags: [kubernetes, securitycontext, psp, psa, selinux, 用户命名空间]
sources: [kubernetes/objects/security-context.md]
---

# 源摘要：SecurityContext

feisky handbook `/concepts/objects/security-context.md` 的要点（深入页见 [[entities/SecurityContext]]）。

## 核心

- SecurityContext 限制不可信容器的行为，保护宿主与其他容器。两级：
  - **容器级**（仅该容器，不含卷）：privileged、capabilities、`allowPrivilegeEscalation`、`readOnlyRootFilesystem`、runAsUser 等。
  - **Pod 级**（含所有容器和卷）：`fsGroup`、`supplementalGroups`、`seLinuxOptions`。

## supplementalGroupsPolicy（v1.33 Beta）

- 控制补充组计算：**Merge**（默认，合并镜像 `/etc/group`）/ **Strict**（只用 fsGroup/supplementalGroups/runAsGroup 指定的，忽略镜像隐式 GID，更安全）。需 containerd 2.0+/CRI-O 1.31+。

## 用户命名空间（v1.33 默认启用）

- `hostUsers: false`：容器内 UID 映射到主机非特权用户——容器内 root 也无法危害主机，降容器逃逸风险。需 Linux 6.3+（推荐）、containerd 2.0+、idmap 挂载；NFS 卷暂不支持。

## SELinux / Capabilities

- `seLinuxOptions` 设进程安全策略；Linux capabilities 按需 add/drop（如 drop ALL + 只加必需）。

## 勘误 / staleness

- ⚠️ **PodSecurityPolicy（PSP）已在 v1.21 弃用、v1.25 移除**——改用内置 **Pod Security Admission（PSA，三档 Privileged/Baseline/Restricted）** 或 OPA/Gatekeeper、Kyverno 等策略引擎。源文档大段 PSP 内容仅作历史参考。
