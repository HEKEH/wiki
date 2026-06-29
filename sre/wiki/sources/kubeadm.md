---
title: 源文档 - kubeadm（部署工具）
date: 2026-06-27
tags: [kubernetes, kubeadm, 部署, source]
sources: [kubernetes/components/kubeadm.md]
---

# 源文档摘要：kubeadm（部署工具）

K8s 官方集群引导工具的部署流程。

## 关键内容

- **前置**：每台机器装容器运行时 + kubelet（kubeadm 靠 kubelet 起控制面）。
- **`kubeadm init`**：预检 → token/CA/证书/kubeconfig → 为控制面组件生成 **static Pod manifest 到 `/etc/kubernetes/manifests/`** → RBAC + 控制节点 taint → 装 kube-proxy/DNS。
- **网络插件**：不内置，需自装 CNI（flannel/calico/weave/bridge）。
- **`kubeadm join`**：下载 CA、签证书、连 apiserver 加节点。
- `kubeadm reset` 卸载。

## 勘误 / 时效性

- 流程仍基本准确（static Pod 引导是现行机制）；具体 CNI 安装 URL/版本已旧，应以各 CNI 官方文档为准。

## 关联

详见实体页 [[entities/kubeadm]] · [[entities/kubelet]]（static Pod 自举） · [[concepts/设计理念]] · [[concepts/集群架构]]
