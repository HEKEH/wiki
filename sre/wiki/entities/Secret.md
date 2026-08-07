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

### 创建（Opaque 示例）

命令式：`kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password='S3cr3t!'`

声明式——**用 `stringData` 写明文，K8s 自动 base64**（比 `data` 手动编码省事）：

```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-secret, namespace: default }
type: Opaque
stringData:                 # 明文写这里,创建时自动转 base64
  username: admin
  password: "S3cr3t!"
# data:                     # 或用 data,但值必须是你已 base64 过的
#   password: UzNjcjN0IQ==
```

### 用法 A：注入环境变量

```yaml
env:
- name: DB_PASSWORD                       # 取单个 key
  valueFrom:
    secretKeyRef: { name: db-secret, key: password }
envFrom:                                  # 或整份注入(key 即变量名)
- secretRef: { name: db-secret }
```

### 用法 B：挂载为卷（tmpfs，每 key 一个文件）

```yaml
containers:
- name: app
  volumeMounts:
  - { name: secret-vol, mountPath: /etc/secret, readOnly: true }
volumes:
- name: secret-vol
  secret:
    secretName: db-secret
    # items: [{ key: password, path: db_pass, mode: 0400 }]   # 可选:挑 key、设文件名/权限
```

挂进去后 `/etc/secret/username`=`admin`、`/etc/secret/password`=`S3cr3t!`。

> **两种用法差异（同 [[entities/ConfigMap]]）**：env 是**启动时一次性读入**，改 Secret 不更新已运行容器、需 `kubectl rollout restart`；**卷挂载的文件会自动更新**（有延迟）。Secret 必须与 Pod **同 namespace**。私有镜像的 `dockerconfigjson` + `imagePullSecrets` 是另一种类型，见 [[entities/Pod]]。

## 静态加密（encryption at rest）

默认 Secret 在 [[entities/etcd]] 里只是 base64。给 [[entities/kube-apiserver]] 配 `EncryptionConfiguration`（`--encryption-provider-config`）可在**写入 etcd 前加密**：

- provider：`identity`（不加密）/ `aescbc` / `aesgcm` / `secretbox` / `kms`（推荐，外部 KMS 托管密钥）。
- 改配置后 `kubectl get secrets -o json | kubectl replace -f -` 可重写以使存量 Secret 全部加密。

## 用 RBAC 限制谁能读

因为 base64 不是加密,**Secret 的"保密"很大程度靠 [[concepts/RBAC]] 限制读取**——最小权限:只把特定 Secret 的读权授给需要它的 [[entities/ServiceAccount]]。

```yaml
# 只允许读名为 db-secret 的这一个 Secret
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: db-secret-reader, namespace: default }
rules:
- apiGroups: [""]                 # core 组(Secret 属 core)
  resources: ["secrets"]
  verbs: ["get"]                  # 只给 get,不给 list
  resourceNames: ["db-secret"]    # 精确到这一个对象
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: db-secret-reader, namespace: default }
subjects:
- { kind: ServiceAccount, name: my-app-sa, namespace: default }   # 授给 Pod 用的 SA
roleRef: { kind: Role, name: db-secret-reader, apiGroup: rbac.authorization.k8s.io }
```

- Pod 经 `serviceAccountName: my-app-sa` 用这个 SA(见 [[entities/Pod]]),就只能读 `db-secret`、读不了别的。
- ⚠️ **`resourceNames` 对 `list`/`watch` 无效**(list 天然返回全部、无法按名过滤)——所以**只给 `get` + `resourceNames`,别给 `list`**,否则等于能列出/读出该 namespace 所有 Secret。
- 内置 ClusterRole 的差异:**`view` 有意排除 Secret**(读不了,防提权);但 **`edit`/`admin` 能访问 Secret**(可借此拿到该 namespace 任意 SA 的权限)——想让人能改别的却读不了 Secret,得自定义 Role,别直接授 `edit`/`admin`。

## 不可变 Secret

`immutable: true`（v1.21 GA）：同 [[entities/ConfigMap]]，禁改、关 watch，降压防误改。

## 勘误 / staleness

- ⚠️ "ServiceAccount 创建时自动建对应 Secret token"——**v1.24 起不再自动创建**长期 token Secret；Pod 改用 **TokenRequest 投射卷**注入**短时效、绑定该 Pod** 的 token（更安全）。见 [[entities/ServiceAccount]]。
- ⚠️ `--experimental-encryption-provider-config` 已更名 `--encryption-provider-config`（早 GA）。

## 相关

- 非敏感配置用：[[entities/ConfigMap]]
- 被谁引用：[[entities/ServiceAccount]]、Pod 的 `imagePullSecrets`（见 [[entities/Pod]]）
- TLS 证书供 Ingress 终止：[[entities/Ingress]]
