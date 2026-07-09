---
title: Secret（资源对象）
date: 2026-06-30
tags: [kubernetes, secret, 敏感数据, 加密, serviceaccount]
sources: [kubernetes/objects/secret.md]
---

# 源摘要：Secret

feisky handbook `/concepts/objects/secret.md` 的要点（深入页见 [[entities/Secret]]）。

## 核心

- 存密码/token/密钥等**敏感数据**，避免写进镜像或 PodSpec。可经 Volume 或环境变量使用，**挂载在 tmpfs（内存）**、Pod 删除即清。

## 三种类型

- **Opaque**：通用 base64 数据（注意：base64 **只是编码、非加密**，加密性很弱）。
- **`kubernetes.io/dockerconfigjson`**：私有 registry 认证，配合 `imagePullSecrets` 拉私有镜像。
- **`kubernetes.io/service-account-token`**：被 ServiceAccount 引用，自动挂到 `/run/secrets/kubernetes.io/serviceaccount`（ca.crt/namespace/token）。

## 进阶

- **静态加密（encryption at rest）**：apiserver 配 `EncryptionConfiguration`，把 Secret 写入 etcd 前加密（aescbc/aesgcm/secretbox/kms），`identity` 为不加密。
- **不可变 Secret**（`immutable: true`，v1.21 GA）：同 ConfigMap，降压防误改。
- vs ConfigMap：Secret 有类型、能被 SA 关联、可做 imagePullSecret、存 tmpfs。

## 勘误 / staleness

- ⚠️ "ServiceAccount 创建时默认创建对应 Secret token"——**v1.24 起不再自动创建**；Pod 改用 **TokenRequest 投射卷**（有时效、绑定 Pod）注入 token。
- ⚠️ `--experimental-encryption-provider-config` 已更名 **`--encryption-provider-config`**（早已 GA）；示例 `kubectl update` 应为 `kubectl replace`。
