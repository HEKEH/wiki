---
title: CustomResourceDefinition (CRD)
date: 2026-06-30
tags: [kubernetes, crd, 自定义资源, operator, 可扩展性, kubebuilder]
sources: [kubernetes/objects/customresourcedefinition.md]
---

# CustomResourceDefinition (CRD)

CRD 让你**不改 K8s 源码**就给 API 加一种新对象类型。声明一个 CRD 后，apiserver 立刻提供该类型的 REST 端点（`/apis/<group>/<version>/<plural>`），可像内置对象一样 `kubectl get/apply`、存进 [[entities/etcd]]、被 RBAC 管控。这是 K8s "平台化/可扩展"的核心机制之一（见 [[concepts/插件机制与可扩展性]]、[[concepts/设计理念]]）。

## 定义要点

- `group` / `versions`（每版本 `served` 是否提供、`storage` 唯一存储版本）。
- `scope`：`Namespaced` 或 `Cluster`。
- `names`：plural / singular / kind / shortNames。

## 最简示例

**① 定义类型**（v1 需 structural schema）：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com          # 必须是 <plural>.<group>
spec:
  group: stable.example.com
  scope: Namespaced
  names: { plural: crontabs, singular: crontab, kind: CronTab, shortNames: ["ct"] }
  versions:
  - name: v1
    served: true                             # 提供该版本 API
    storage: true                            # 存储版本(有且仅一个)
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec: { type: string }
              image:    { type: string }
```

**② 创建实例(CR)并像内置对象一样用**：

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata: { name: my-cron }
spec: { cronSpec: "* * * * */5", image: my-cron-image }
```

```bash
kubectl apply -f crd.yaml && kubectl apply -f my-cron.yaml   # 注册类型 + 存进 etcd
kubectl get ct                 # 像内置对象一样查（还可 describe/edit/delete）
```

> 光有 CRD 只是"能存/查数据"，**不会发生任何行为**——要有行为得配控制器（见下）。

## 关键能力

- **Validation**：基于 OpenAPI v3 schema 校验字段（类型、范围、正则）；v1 要求 **structural schema**（结构化、字段都声明类型）。
- **Subresources**（写在 `versions[].subresources`）：
  - `status: {}` 开启 `/status`：主端点写入忽略 `.status`、`/status` 写入忽略 `.spec`，实现 **spec（用户期望）/ status（控制器实况）分权互不覆盖**；且 `metadata.generation` 只在 spec 变时自增，供控制器用 `status.observedGeneration == generation` 判断"最新期望是否已处理"。
  - `scale` 开启 `/scale`：让 CR 暴露标准 Scale 对象，`kubectl scale` 和 [[entities/Autoscaling]] HPA 无需认识该类型即可伸缩。三个 JSONPath 把统一协议**映射**到你的字段（不创建字段，须已在 schema 声明）：`specReplicasPath`=别人写入的副本旋钮（如 `.spec.replicas`）、`statusReplicasPath`=控制器上报的实际副本数（HPA 据此算比例）、`labelSelectorPath`=(可选)供 HPA 找 Pod 读指标的选择器串。
- **Finalizer**（`metadata.finalizers`）：删除时先置 `deletionTimestamp` 触发控制器做清理、移除自己的 finalizer 后才真正删——**异步预删除钩子**。
- **Categories**：归组以便 `kubectl get <category>`。

## 多版本与版本演进

`versions` 是列表,可同时声明多个版本,两个字段含义不同:

- **`served`**:该版本的 REST 端点**是否开放**。多个版本可同时 `served: true`,apiserver 负责在它们间转换,老客户端不受影响。`served: false` 是**退役动作**(端点关闭),不是升级动作。
- **`storage`**:对象**按哪个版本落库**。**任意时刻有且仅一个** `storage: true`(写两个直接报错)。用别的 served 版本写入时,apiserver 转成 storage 版本再存。

> ⚠️ **翻转 `storage` 不会重编码 etcd 里的存量对象**——已存在的旧对象仍是旧格式,`status.storedVersions` 记录用过的存储版本。因此想彻底删旧版本,完整顺序是:① 加新版 `served: true`(schema 不同则配 **conversion webhook**,相同用 `None`)→ ② 新版 `storage: true`、旧版改 `false` → ③ **迁移存量**(逐个读出再写回,或 StorageVersionMigration)→ ④ 从 `status.storedVersions` 移除旧版 → ⑤ 旧版 `served: false`、最终删除。

## CRD + 控制器 = Operator 模式

CRD 只声明"**数据长什么样**"；要让它"**有行为**"，得配一个**自定义控制器** watch 它并 reconcile 到期望状态（套用 [[concepts/控制平面与控制循环]] 的同一范式）。CRD + 控制器即 **Operator**，把运维知识（如"建一个数据库集群"）编码成声明式对象。脚手架：`kubebuilder`、sample-controller。

**控制器怎么"跑起来"**：它**不进 [[entities/kube-controller-manager]]**（那里只有编译死的内置控制器），而是你**独立部署的一个普通 Pod**——`Deployment` + [[entities/ServiceAccount]] + [[concepts/RBAC]]，靠 SA token 连 apiserver 主动 watch，**无需向 K8s"注册"**（apiserver 只是被动提供 API）。CRD + RBAC + 控制器 Deployment 通常打包一起 `apply`。

**控制器既 watch 又执行**（没有独立的"执行者 Pod"，控制器自己就是执行者）：

- 动作是**造 K8s 对象**（如建 Job/Pod）→ 经 apiserver 写 [[entities/etcd]]，最终由 [[entities/kubelet]] 在节点真正起容器。
- 动作是**外部副作用**（如 cert-manager 向 Let's Encrypt 签证书、云 Operator 调云 API）→ **控制器 Pod 自己直接发请求**，再把结果写回集群（如一个 [[entities/Secret]]）。
- 全程只 watch apiserver、**从不直接碰 etcd**。

## 生态中的著名 CRD

几乎都由 Operator/生态项目交付：**cert-manager**（`Certificate`/`Issuer`）、**Prometheus Operator**（`ServiceMonitor`）、**Argo CD**（`Application`）、**Istio**（`VirtualService`）、[[entities/GatewayAPI]]（`Gateway`/`HTTPRoute`）、**KEDA**（`ScaledObject`）、**Velero**（`Backup`）、**KubeVirt**（`VirtualMachine`）等。

> 注意：**VPA（`VerticalPodAutoscaler`）本身就是 CRD**（故需单独安装，见 [[entities/Autoscaling]]），而 HPA 是内置对象——这也是二者部署方式不同的根源。

## 勘误 / staleness

> ⚠️ 勘误：API 现为 **`apiextensions.k8s.io/v1`（GA v1.16，v1beta1 于 v1.22 移除）**；源文档的 `v1beta1`、单数 `version` 字段、各 alpha 门控均已过时（structural schema 现为强制）。CRD 取代了更早已废弃的 TPR（ThirdPartyResource）。

## 相关

- 可扩展性总览（CRI/CNI/CSI + CRD/Webhook）：[[concepts/插件机制与可扩展性]]
- 控制器范式：[[concepts/控制平面与控制循环]]
- 名词式可组合 API 的设计原则：[[concepts/设计理念]]
