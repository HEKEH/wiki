---
title: Secret
date: 2026-06-30
tags: [kubernetes, secret, 敏感数据, 加密, serviceaccount, imagepullsecret]
sources: [kubernetes/objects/secret.md]
---

# Secret

Secret 存放密码、token、密钥等**敏感数据**，避免把它们硬编码进镜像或 PodSpec。与 [[entities/ConfigMap]] 形态相同（key-value、属某 namespace、可挂卷或注入 env），但有类型、可被 ServiceAccount 关联、挂载在 **tmpfs（内存）**、Pod 删除即清。

## 三种类型

| 类型 | 用途 |
| --- | --- |
| **Opaque** | 通用敏感数据（密码/密钥），value 为 base64 |
| **`kubernetes.io/dockerconfigjson`** | 私有镜像仓库认证，配合 Pod 的 `imagePullSecrets` 拉私有镜像 |
| **`kubernetes.io/service-account-token`** | 被 ServiceAccount 引用的 API 访问令牌 |

> ⚠️ **base64 不是加密**，只是编码（`base64 --decode` 即还原）。Secret 的"保密"靠 RBAC 限制读取 + 下面的静态加密，而非编码本身。

## 使用方式

- **Volume 挂载**：`volumes.secret`，每个 key 成一个文件（`items` 可挑 key、设 `mode`/`path`）；挂载在 tmpfs，Pod 删即清（但节点上 `/var/lib/kubelet/pods/.../volumes/...~secret/` 运行期可见）。
- **环境变量**：`env.valueFrom.secretKeyRef`。

## 静态加密（encryption at rest）

默认 Secret 在 [[entities/etcd]] 里只是 base64。给 [[entities/kube-apiserver]] 配 `EncryptionConfiguration`（`--encryption-provider-config`）可在**写入 etcd 前加密**：

- provider：`identity`（不加密）/ `aescbc` / `aesgcm` / `secretbox` / `kms`（推荐，外部 KMS 托管密钥）。
- 改配置后 `kubectl get secrets -o json | kubectl replace -f -` 可重写以使存量 Secret 全部加密。

## 不可变 Secret

`immutable: true`（v1.21 GA）：同 [[entities/ConfigMap]]，禁改、关 watch，降压防误改。

## 勘误 / staleness

- ⚠️ "ServiceAccount 创建时自动建对应 Secret token"——**v1.24 起不再自动创建**长期 token Secret；Pod 改用 **TokenRequest 投射卷**注入**短时效、绑定该 Pod** 的 token（更安全）。见 [[entities/ServiceAccount]]。
- ⚠️ `--experimental-encryption-provider-config` 已更名 `--encryption-provider-config`（早 GA）。

## 相关

- 非敏感配置用：[[entities/ConfigMap]]
- 被谁引用：[[entities/ServiceAccount]]、Pod 的 `imagePullSecrets`（见 [[entities/Pod]]）
- TLS 证书供 Ingress 终止：[[entities/Ingress]]
