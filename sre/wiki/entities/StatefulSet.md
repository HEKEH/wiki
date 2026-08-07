---
title: StatefulSet
date: 2026-06-30
tags: [kubernetes, statefulset, 有状态, headless-service, pvc, 有序]
sources: [kubernetes/objects/statefulset.md]
---

# StatefulSet

StatefulSet 是管理**有状态应用**的控制器（与为无状态设计的 [[entities/Deployment]]/[[entities/ReplicaSet]] 相对）。它给每个 [[entities/Pod]] 一个**稳定且可单独寻址的身份**——把"牲畜"变"宠物"（选型与"宠物 vs 牲畜"模型见 [[concepts/工作负载控制器]]）。

## 三种"稳定"

1. **稳定网络标识**：Pod 名有序固定 `web-0/1/2`，重建/漂移不变；配合 **Headless Service（`clusterIP: None`）** 得到固定 DNS：
   ```text
   <statefulSetName>-{0..N-1}.<serviceName>.<namespace>.svc.cluster.local
   ```
   （Headless Service 见 [[entities/Service]]、[[entities/CoreDNS]]）
2. **稳定持久存储**：经 `volumeClaimTemplates` 为**每个 Pod 各创建一份 PVC**（如 `www-web-0`），重建后挂回原盘（PVC/PV 见 [[concepts/Volume存储]]）。
3. **有序部署/扩展（0→N-1）、有序收缩/删除（N-1→0）**。

## 组成三件套

- 定义 DNS 域的 **Headless Service**（须先于 StatefulSet 创建）
- **volumeClaimTemplates**（每副本独立存储）
- StatefulSet 本体（`serviceName` 指向上面的 headless service）

## 定义示例（Headless Service + StatefulSet）

```yaml
# ① Headless Service —— 它的 name 就是 per-pod DNS 里的"域"那一段
apiVersion: v1
kind: Service
metadata: { name: nginx, namespace: default, labels: { app: nginx } }
spec:
  clusterIP: None                # ← 关键:headless(不分配 ClusterIP、不负载均衡)
  selector: { app: nginx }       # 须与下面 template 的 labels 一致
  ports: [{ name: web, port: 80 }]
  # publishNotReadyAddresses: true   # 集群 bootstrap 需成员未 Ready 就互相解析时开
---
# ② StatefulSet —— serviceName 必须等于上面 Service 的 name
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: web, namespace: default }
spec:
  serviceName: nginx             # ← 对齐 Service 名,才生成 per-pod DNS
  replicas: 3
  selector: { matchLabels: { app: nginx } }
  template:
    metadata: { labels: { app: nginx } }    # ← 须与 Service selector 匹配
    spec:
      containers:
      - { name: nginx, image: nginx, ports: [{ containerPort: 80, name: web }] }
  volumeClaimTemplates:          # 每副本各一份 PVC:www... 见"三种稳定"②
  - metadata: { name: www }
    spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 1Gi } } }
```

生成的 DNS：`nginx.default.svc.cluster.local` 返回全部 Pod IP；`web-0.nginx.default.svc.cluster.local`（及 web-1/2）精确到单个 Pod。

**三处必对齐**（错一个 per-pod DNS 就不生成）：① `clusterIP: None`；② StatefulSet `serviceName` == Service `name`；③ Service `selector` == Pod `template` labels。另：**先建 Service 再建 StatefulSet**；集群型应用 bootstrap 时成员未 Ready 就要互相解析，开 `publishNotReadyAddresses: true`。

## 更新策略 `.spec.updateStrategy`

- **RollingUpdate**（默认）：按**逆序**（N-1→0）逐个删建，等 Ready 再下一个。
  - **Partitions**：`rollingUpdate.partition=K` → 仅序号 **≥ K** 的 Pod 更新，其余保持旧版——可做**金丝雀/分批灰度**。
- **OnDelete**：改模板后不动旧 Pod，**手动删除**某 Pod 才用新模板重建。

## Pod 管理策略 `.spec.podManagementPolicy`

- **OrderedReady**（默认）：按序创建，等前一个 Ready 才建下一个。
- **Parallel**：并行创建/删除（仍保留稳定身份，只是不等待）。

## 注意

- 删除 StatefulSet **不会删 PVC**（保数据安全），数据不用了需手动删 PVC。
- 典型用途：MySQL/PostgreSQL、Zookeeper、[[entities/etcd]] 等集群型有状态服务。

> 历史：v1.3 以 Alpha 版 **PetSet** 发布，v1.5 重命名为 StatefulSet，**v1.9 GA**。
> ⚠️ 勘误：源称"OnDelete 为默认更新策略"已过时——**apps/v1 默认 RollingUpdate**（OnDelete 是兼容 v1.6 的旧默认）；示例的 `k8s.gcr.io`/`kube-dns` 现为 `registry.k8s.io`/CoreDNS。

## 相关

- 控制器选型：[[concepts/工作负载控制器]]
- 稳定身份为何对集群型应用不可或缺：见 [[concepts/工作负载控制器]] StatefulSet 一节
