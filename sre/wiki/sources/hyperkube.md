---
title: 源文档 - hyperkube
date: 2026-06-27
tags: [kubernetes, hyperkube, source, 已废弃]
sources: [kubernetes/components/hyperkube.md]
---

# 源文档摘要：hyperkube

极短的一页：hyperkube 是把多个 K8s 组件打包进一个二进制的 all-in-one binary。

## 关键内容

- hyperkube 可通过子命令启动 kubelet / apiserver / controller-manager / scheduler / proxy / kubectl / federation-*，常用于容器镜像。
- 每个发布版会同时发布含 hyperkube 的 docker 镜像（如 `gcr.io/.../hyperkube:v1.6.4`）。

## 勘误 / 时效性

- ⚠️ **hyperkube 已在 v1.17 移除**（停止发布官方 hyperkube 镜像）。现各组件以独立二进制/镜像分发；本页仅作历史参考。

## 关联

[[entities/Kubernetes]]（核心组件清单）
