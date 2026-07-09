---
title: RBAC（基于角色的访问控制）
date: 2026-07-01
tags: [kubernetes, rbac, 授权, 访问控制, 安全, role, rolebinding]
sources: [kubernetes/objects/serviceaccount.md]
---

# RBAC（Role-Based Access Control）

RBAC 是 K8s 的**授权机制**——决定"**某个身份能对哪些资源做哪些操作**"。它是 [[entities/kube-apiserver]] 访问控制三关卡里的**授权**那一关：认证（[[entities/ServiceAccount]]/证书）回答"**你是谁**"，RBAC 回答"**你能做什么**"，准入控制再回答"**这次操作是否合规**"。RBAC 自 v1.8 起 GA 且为默认启用（`rbac.authorization.k8s.io/v1`）。

## 四个对象（2×2）

分成"**定义权限**"和"**把权限授予谁**"两对，各有 namespace 级和集群级：

| | 定义权限（能做什么） | 绑定主体（授予谁） |
| --- | --- | --- |
| **namespace 级** | **Role** | **RoleBinding** |
| **集群级** | **ClusterRole** | **ClusterRoleBinding** |

- **Role / ClusterRole**：一组规则的集合（如"能 get/list/watch pods"）。
- **RoleBinding / ClusterRoleBinding**：把某个 Role/ClusterRole **授予**一批主体。

## 三个要素

1. **主体（subject，谁）**：`User` / `Group`（人，来自外部认证，K8s 不存 user 对象）、`ServiceAccount`（机器身份，见 [[entities/ServiceAccount]]）。
2. **规则（rule，能做什么）**：`apiGroups` + `resources` + `verbs` 三者拼出一条权限。
3. **默认拒绝、只能"加允许"**：RBAC 是**纯白名单**，没有 deny 规则（和 [[entities/NetworkPolicy]] 同思路）——没显式授予就没权限。多条规则、多个 binding 的权限**叠加取并集**。

## 逐行拆解一个例子

目标：**让 `default` 这个 ServiceAccount 能在本 namespace 里只读 Pod**。拆成"定义权限"和"授权给谁"两步。

### ① Role —— 定义"能做什么"

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader           # 这个权限包的名字（下面 RoleBinding 按名引用它）
  namespace: default         # Role 是 namespace 级的，只在 default 内有效
rules:
- apiGroups: [""]            # 资源所属 API 组；"" 代表 core 组（Pod/Service/ConfigMap…）
  resources: ["pods"]        # 作用于哪种资源（用复数名，可多个）
  verbs: ["get", "list", "watch"]   # 允许的动作（此处仅只读）
```

- **`apiGroups: [""]`**：空字符串代表 **core（核心）组**——Pod、Service、ConfigMap、Node 这些最老的内置资源（`apiVersion` 就是光秃秃的 `v1`）。Deployment 属 `apps` 组，就写 `["apps"]`。
- **`verbs`**：常见 `get`（读单个）/`list`（列全部）/`watch`（监听变更）/`create`/`update`/`patch`/`delete`。只给前三个 → 只能看、不能改删。
- 此时**还没有任何人拿到权限**，Role 只是把"权限包"放在那儿。

### ② RoleBinding —— 把权限"授予谁"

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:                    # 授予给谁（主体，可多个）
- kind: ServiceAccount
  name: default
  namespace: default
roleRef:                     # 授予哪个权限包（一旦创建不可改）
  kind: Role
  name: pod-reader           # ← 对应上面 Role 的 metadata.name
  apiGroup: rbac.authorization.k8s.io
```

- **`subjects`**：谁得到权限。这里是 `default` SA，也可是 `User`/`Group`。
- **`roleRef`**：引用哪个权限包，按 `kind`+`name` 指向 `pod-reader`。

**两个 `apiGroup` 别混**（API group = 资源类型的分组，即对象 `apiVersion: <group>/<version>` 里的 group）：

- **`rules[].apiGroups`** 指**被授权的资源**在哪个 group——随资源变：`""`(core: Pod/Service…)、`apps`(Deployment)、`gateway.networking.k8s.io`(Gateway)…
- **`roleRef.apiGroup`** 指**被引用的 Role 对象**在哪个 group——因 Role/ClusterRole 定义于 RBAC，**固定是 `rbac.authorization.k8s.io`**。
- **`subjects[].apiGroup`** 同理指主体对象的 group：`User`/`Group` 写 `rbac.authorization.k8s.io`，而 **`ServiceAccount` 写 `""`**(它是 core 组的真实对象)。

> **合起来**：凡以 `default` SA 身份访问 apiserver 的请求（如用了该 SA 的 Pod 里跑的程序），就能在 default namespace 里 get/list/watch pods，别的一概不行。

## 常见组合

- **Role + RoleBinding**：某 namespace 内的权限。
- **ClusterRole + ClusterRoleBinding**：全集群权限。
- **ClusterRole + RoleBinding**（常用技巧）：**复用**一份 ClusterRole 定义，但只在指定 namespace 授予——避免每个 namespace 重写 Role。RoleBinding 的 `roleRef.kind` 可为 `Role` 或 `ClusterRole`。
- 集群级 ClusterRole 还能授"非资源型"端点（`nonResourceURLs`，如 `/healthz`）和跨 namespace 资源。

## 内置 ClusterRole

开箱即用的四个：`cluster-admin`（超管，一切权限）、`admin`（namespace 内近乎全权）、`edit`（读写大部分资源、不能改 RBAC）、`view`（只读）。通常配 RoleBinding 授给某 namespace 的用户/SA。

## 两步分离的好处

- 一个 Role 可被多个 RoleBinding 复用、授给不同主体；改权限只改 Role，换人只改 RoleBinding。
- 这也解释了"**自定义控制器要配 RBAC**"（见 [[entities/CustomResourceDefinition]]）：控制器的 [[entities/ServiceAccount]] 得经 ClusterRole/Binding 拿到"watch 某 CRD、创建 Job"等权限，否则连不上也改不动。

> 一句话：**RBAC = 用 Role/ClusterRole 定义"能对什么资源做什么动词"，再用 RoleBinding/ClusterRoleBinding 授予 User/Group/ServiceAccount；默认拒绝、只能加允许，是 apiserver 的授权关。**

## 相关

- 机器身份（RBAC 的常见授予对象）：[[entities/ServiceAccount]]
- 访问控制全链路（认证→授权→准入）：[[entities/kube-apiserver]]
- 同为白名单思路的网络隔离：[[entities/NetworkPolicy]]
- 自定义控制器的权限配置：[[entities/CustomResourceDefinition]]
