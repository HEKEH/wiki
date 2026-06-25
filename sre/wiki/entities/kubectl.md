---
title: kubectl
date: 2026-06-23
tags: [kubernetes, kubectl, 命令行]
sources: [kubernetes/kubernetes 101.md, kubernetes/kubernetes 201.md]
---

# kubectl

kubectl 是 Kubernetes 的命令行工具，用于创建、查询、操作集群中的资源。许多命令与 Docker 一一对应，便于上手。

## 定位与边界

- **是 apiserver 的客户端之一**：处于管理层、集群之外，本质是封装了 K8s REST API 的瘦客户端；**不在 [[entities/Pod]]→[[entities/Service]] 的应用数据路径上**。GitOps（Argo/Flux）、Operator、CI 等自动化常**直接调 API 而绕开 kubectl**——kubectl 主要给人做交互/调试。
- **不创建集群**：集群由 kubeadm / minikube / 云厂商工具启动（apiserver 起来后 kubectl 才能连）；但集群起来后常用 kubectl 验证（`get nodes`）和装 CNI/插件补全可用性。
- **操作的是同一集群里的对象**：多次 `create` 只是在**同一集群**里建多个对象（非多个集群），它们默认网络互通、可经 label/DNS 互相关联。同 `(namespace, 类型, 名字)` 不能重复创建（报 `AlreadyExists`）；声明式更新宜用 `kubectl apply -f`（有则更新、无则创建）。

## 与 Docker 对应的常用命令

| kubectl | 类比 docker | 作用 |
| --- | --- | --- |
| `kubectl run` | `docker run` | 创建容器（实为创建 [[entities/Deployment]] 管理的 Pod） |
| `kubectl get` | `docker ps` | 查询资源列表 |
| `kubectl describe` | `docker inspect` | 获取资源详细信息 |
| `kubectl logs` | `docker logs` | 获取容器日志 |
| `kubectl exec` | `docker exec` | 在容器内执行命令 |

## 资源管理（声明式）

- `kubectl create -f file.yaml` —— 用 manifest 创建资源（比 `kubectl run` 功能更全）。
- `kubectl expose` —— 创建 [[entities/Service]] 暴露应用。
- `kubectl scale --replicas=N` —— 扩缩容。
- `kubectl set image` / `kubectl set resources` —— 更新镜像 / 设置[[concepts/资源限制]]。
- `kubectl rollout status|history|undo` —— 查看 / 回滚滚动升级。
- `kubectl edit` —— 在线编辑 manifest。
- `kubectl cluster-info` —— 查看集群信息。

详见 [[concepts/扩缩容与滚动升级]]。
