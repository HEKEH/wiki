---
title: DaemonSet
date: 2026-06-30
tags: [kubernetes, daemonset, 节点守护, 日志, 监控]
sources: [kubernetes/objects/daemonset.md]
---

# DaemonSet

DaemonSet 保证**每个（或选定的）[[entities/Node]] 上运行一个 [[entities/Pod]] 副本**。关注点在**节点覆盖**而非业务副本数——节点加入集群即自动补一个，节点移除即回收。

## 典型用途（每节点支撑服务）

- **日志收集**：fluentd、logstash
- **监控**：Prometheus node-exporter、collectd
- **系统程序**：[[entities/kube-proxy]]、CNI、ceph/glusterd 等

> DaemonSet **忽略 Node 的 unschedulable 状态**——即便节点被标记不可调度，守护 Pod 仍照常下发（这类基础服务必须无差别覆盖）。

## 示例定义

以 node-exporter 为例,把常用属性都放进来(实际按需取用):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:                       # 必填,须匹配 template 的 labels
    matchLabels:
      app: node-exporter
  updateStrategy:                 # 更新策略(默认即 RollingUpdate)
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1           # 一次最多几个节点上的 Pod 不可用
  template:
    metadata:
      labels:
        app: node-exporter        # 必须与上面 selector 对应
    spec:
      nodeSelector:               # ① 简单筛选:只在带此 label 的节点跑
        node-type: worker
      affinity:
        nodeAffinity:             # ② 更强表达:排除带 GPU 的节点
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: hardware
                operator: NotIn   # In/NotIn/Exists/DoesNotExist/Gt/Lt
                values: ["gpu"]
      tolerations:                # ③ 容忍控制节点 taint 才能覆盖到它
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        resources:                # 建议给基础服务设资源上下限
          requests: { cpu: 50m, memory: 64Mi }
          limits:   { cpu: 200m, memory: 128Mi }
```

- **无 `replicas`**:副本数不由你定,而是"**每个符合条件的 [[entities/Node]] 一个**",节点增减自动增减 Pod。这是与 [[entities/Deployment]] 结构上最大的区别(其余 `selector`+`template` 写法一致)。
- **`nodeSelector` vs `nodeAffinity`**:前者是简单等值筛选(所有 label 必须命中);后者支持 `In/NotIn/Exists/Gt/Lt` 等表达式,能做"排除/择一/软偏好"(`preferredDuringScheduling…` 为软约束)。二者都在 `template.spec` 里,即"这些 Pod 允许落在哪些节点";DaemonSet 再在这些候选节点上**逐一铺一个**。

## 限定运行节点

- `nodeSelector`：调度到匹配 label 的节点（见 [[concepts/Label与Selector]]）。
- `nodeAffinity` / `podAffinity`：更丰富的选择（见 [[entities/kube-scheduler]]）。
- 若节点带 taint（如控制节点），DaemonSet 需相应 **toleration** 才能上（见 [[entities/kube-scheduler]] 污点与容忍）。

## 更新与回滚

- `.spec.updateStrategy.type`：**RollingUpdate** / **OnDelete**；RollingUpdate 可设 `maxUnavailable`、`minReadySeconds`。
- 支持 `kubectl rollout history/undo/status`（v1.7+）。

## 与静态 Pod 的区别

也可用**静态 Pod**（kubelet `--pod-manifest-path` 目录）在每台机器跑指定 Pod，但静态 Pod 由各节点 kubelet 本地管理、不能经 API Server 删除（删 manifest 文件才行），无统一滚动更新——常规每节点服务用 DaemonSet 更可控（见 [[entities/kubelet]] 静态 Pod）。

> ⚠️ 勘误：源称"OnDelete 为默认更新策略"已过时——**apps/v1 默认 RollingUpdate**；示例容忍 `node-role.kubernetes.io/master` 现应为 `.../control-plane`（见 [[entities/kubeadm]]）。

## 相关

- 控制器选型：[[concepts/工作负载控制器]]
- 节点组件本身多以 DaemonSet 部署：[[entities/Node]]
