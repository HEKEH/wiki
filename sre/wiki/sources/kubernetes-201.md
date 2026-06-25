---
title: Kubernetes 201（源文档摘要）
date: 2026-06-23
tags: [kubernetes, 扩缩容, 滚动升级, 资源限制, 健康检查]
sources: [kubernetes/kubernetes 201.md]
---

# Kubernetes 201（源文档摘要）

来源：`raw/kubernetes/kubernetes 201.md` ｜ 原文：<https://kubernetes.feisky.xyz/introduction/201>

进阶操作，围绕 [[entities/Deployment]] 的运维能力：

- **扩展应用**：改 `replicas` 动态扩缩容（`kubectl scale`），伸缩的容器自动加入/移出 Service。详见 [[concepts/扩缩容与滚动升级]]。
- **滚动升级**：逐个替换容器实现无中断升级；`kubectl rolling-update` 仅针对 ReplicationController，Deployment 用 RollingUpdate 策略 + `kubectl set image` / `rollout`，支持回滚。
- **资源限制**：通过 cgroups 限制容器 CPU/内存（`kubectl set resources` 或 `resources.limits`）。详见 [[concepts/资源限制]]。
- **健康检查**：LivenessProbe（不健康则重建容器）与 ReadinessProbe（未就绪则不接流量），支持 exec/tcpSocket/http。详见 [[concepts/健康检查]]。
