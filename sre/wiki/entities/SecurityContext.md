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

## 配置示例（把加固惯用法全用上）

```yaml
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  securityContext:                    # ── Pod 级:所有容器 + 卷 ──
    runAsNonRoot: true                # 禁止以 root 跑(root 镜像会被拒启动)
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000                     # 挂载卷的属组,让非 root 进程能读写卷
    seccompProfile: { type: RuntimeDefault }   # 启用默认 seccomp,过滤危险 syscall
  containers:
  - name: app
    image: myapp
    securityContext:                  # ── 容器级:仅此容器,同名字段覆盖 Pod 级 ──
      privileged: false
      allowPrivilegeEscalation: false # 禁止 setuid 等提权
      readOnlyRootFilesystem: true    # 根文件系统只读,防篡改
      capabilities:
        drop: ["ALL"]                 # 丢弃全部 capability
        add:  ["NET_BIND_SERVICE"]    # 只加回确实需要的(如绑 <1024 端口)
    volumeMounts:
    - { name: tmp, mountPath: /tmp }  # 根只读时,需写目录用可写卷补上
  volumes:
  - name: tmp
    emptyDir: {}
```

| 字段 | 作用 |
| --- | --- |
| `runAsUser`（如 1000） | 进程的 **UID**（以谁跑;新建文件的属主）。`runAsNonRoot: true` 则**拒绝** root 镜像启动 |
| `runAsGroup`（如 3000） | 进程的**主组 GID**（新建文件的属组）。⚠️ **不设默认是 0（root 组）**,即使 UID 非 root——要彻底非 root 须显式设 |
| `fsGroup`（Pod 级,如 2000） | 给**挂载卷**设属组 + 塞进进程附加组 → **非 root 进程才能读写卷**（卷默认属 root、否则写不了）。大卷可配 `fsGroupChangePolicy: OnRootMismatch` |
| `seccompProfile: RuntimeDefault` | 启用运行时默认 seccomp,拦危险 syscall |
| `allowPrivilegeEscalation: false` | 禁止 setuid 等提权 |
| `readOnlyRootFilesystem: true` | 根只读,篡改不了二进制/配置 |
| `capabilities.drop:[ALL]`+`add` | 先丢光 root 全部能力,再按需加回极少数 |

- **层级覆盖**：容器级同名字段**优先于** Pod 级；`fsGroup`/`seLinuxOptions` 只在 Pod 级。
- **`readOnlyRootFilesystem` 的配套**：应用要写 `/tmp`/缓存等,须用 `emptyDir` 把可写路径挂上,否则写不了而崩。
- 这套基本满足 PSA 的 **Restricted** 档（见下 PSP→PSA）。

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
