---
title: kubeadm
date: 2026-06-27
tags: [kubernetes, kubeadm, 部署, 集群引导, static-pod]
sources: [kubernetes/components/kubeadm.md]
---

# kubeadm

kubeadm 是 Kubernetes 官方主推的**集群部署/引导工具**，把"装出一个最小可用集群"标准化为几条命令。它是 [[concepts/设计理念]] 中**自举（self-hosting）/ static Pod 引导**的典型落地（见 [[entities/kubelet]] 静态 Pod 一节）。

## 前置条件

每台机器先装好**容器运行时**（containerd 等）和 **[[entities/kubelet]]**——因为 kubeadm 依赖 kubelet 以 static Pod 方式拉起控制平面组件。

## 安装

kubeadm 是个**独立 CLI 二进制,要单独装**——运行它时集群还不存在,只能从 OS 包管理器预装(先有鸡先有蛋)。各工具谁装在哪:

| 工具 | 每节点都装? | 说明 |
| --- | --- | --- |
| **[[entities/kubelet]]** | ✅ 必须(所有节点) | 节点 agent,**常驻** systemd 服务 |
| **kubeadm** | ✅ 是(所有要 init/join 的节点) | **一次性** CLI,仅 init/join/upgrade/reset 时运行,非常驻 |
| **[[entities/kubectl]]** | ❌ 不必 | 客户端,只装在**发命令的机器**(管理机或某控制节点);worker 无 kubeconfig 凭证,装了也用不上 |

> 官方文档常把 `kubelet kubeadm kubectl` 三件套一起装,只是**图省事**,不是技术要求。**例外**:GKE/EKS/AKS 等**托管集群**由云厂商管控制平面,**完全不用 kubeadm**——它只用于自建/on-prem 集群。

## 装控制平面：`kubeadm init`

```bash
kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version stable
```

自动完成：系统预检 → 生成 token → 生成自签名 CA 与各组件证书 → 生成 kubeconfig → **为 apiserver/controller-manager/scheduler/[[entities/etcd]] 生成 static Pod manifest 放到 `/etc/kubernetes/manifests/`**（kubelet 一启动就把它们跑起来，解决"谁来起 apiserver"的循环依赖）→ 配置 RBAC、给控制节点打 taint 只跑控制平面 → 部署 kube-proxy、CoreDNS 等附加组件。

## 控制节点上跑什么（taint 的作用）

`init` 给控制节点打的 taint 是 `node-role.kubernetes.io/control-plane:NoSchedule`（旧版 `.../master`）。效果：

- **业务 Pod 默认不会上控制节点**——没带对应 toleration 就被挡住，目的是**隔离/保护控制平面**(别让业务负载和 apiserver/etcd 抢资源)。
- 但控制节点上**仍有 Pod**：① 控制平面静态 Pod(apiserver/controller-manager/scheduler/[[entities/etcd]])；② **带容忍的系统 DaemonSet**(kube-proxy、CNI、监控/日志 agent)照常跑。
- **例外**：单节点/开发集群(minikube、kind)会**去掉 taint**(`kubectl taint nodes --all node-role.kubernetes.io/control-plane-`)，否则没处跑业务 Pod；也可给特定 Pod 加 toleration 定向投放(放行≠吸引，见 [[entities/kube-scheduler]])。

## 网络插件

kubeadm **不内置 CNI**，默认让 kubelet 用 CNI 但需用户自己装网络插件（flannel / calico / weave 或手写 CNI bridge 配置），见 [[concepts/插件机制与可扩展性]]、[[concepts/Pod网络模型]]。

## 加节点：`kubeadm join`

```bash
kubeadm join --token <token> <master_ip>
```

从 apiserver 下载 CA → 本地生成证书并请求签名 → 配置 kubelet 连上 apiserver。

## 其他

- `kubeadm reset`：卸载/清理。
- `kubeadm token list`：管理引导 token。

## 相关

- 引导/自举与 static Pod：[[entities/kubelet]]、[[concepts/设计理念]]
- 集群组成：[[concepts/集群架构]]
- all-in-one 二进制（旧）：[[entities/Kubernetes]] 组件清单
