---
title: 源文档 - kubelet
date: 2026-06-27
tags: [kubernetes, kubelet, static-pod, CRI, 驱逐, source]
sources: [kubernetes/components/kubelet.md]
---

# 源文档摘要：kubelet

详述节点/Pod 管理、static Pod、健康检查、CRI、驱逐与内部模块。

## 关键内容

- **节点管理**：`--register-node` 自注册、定期上报 NodeStatus。
- **Pod 清单来源**：apiserver / 本地文件（`/etc/kubernetes/manifests/`）/ HTTP；创建时先起 **pause 容器**接管 Pod 网络，再挂卷、拉 Secret、起容器。
- **Static Pod / Mirror Pod**：非 apiserver 创建的 Pod，kubelet 在 apiserver 建只读 mirror pod 反映其状态。
- **健康检查**：Liveness/Readiness 探针（Exec/TCP/HTTP）。
- **驱逐**：memory/nodefs/imagefs 信号，软/硬驱逐，用户 Pod 按 BestEffort→Burstable→Guaranteed 顺序驱逐。
- **CRI**：kubelet 是客户端、运行时实现服务端（gRPC，RuntimeService/ImageService）。
- **内部模块**：syncLoop、PLEG、Volume Manager、cAdvisor、网络插件等；Kubelet API（10250 等）。

## 勘误 / 时效性

- ⚠️ 大量以 Docker/dockershim 为默认运行时——**dockershim 已 v1.24 移除**；`--network-plugin` 已移除（CNI 唯一）；rkt 已停。
- ⚠️ Heapster 已废弃（改 metrics-server）；cAdvisor `4194` 端口已移除；`--cadvisor-port=0` 等启动参数过时。
- syncLoop / PLEG / CRI 客户端-服务端模型仍准确。

## 关联

详见实体页 [[entities/kubelet]] · [[concepts/插件机制与可扩展性]] · [[concepts/健康检查]] · [[concepts/资源限制]]
