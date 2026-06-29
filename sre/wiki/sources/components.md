---
title: 源文档 - 核心组件总览（components）
date: 2026-06-27
tags: [kubernetes, 组件, 端口, 版本策略, source]
sources: [kubernetes/components/components.md]
---

# 源文档摘要：核心组件总览（components）

`kubernetes-handbook` 的核心组件总览页，是 components 子目录的入口。

## 关键内容

- **七大核心组件**：[[entities/etcd]]（状态）、[[entities/kube-apiserver]]（唯一入口）、[[entities/kube-controller-manager]]（维护状态）、[[entities/kube-scheduler]]（调度）、[[entities/kubelet]]（容器生命周期 + CVI/CNI）、Container Runtime（CRI）、[[entities/kube-proxy]]（Service 负载均衡）。
- **组件通信**：只有 apiserver 直接操作 etcd；其余组件经 apiserver 的 REST/watch API 交互；apiserver 也会反向调用 Kubelet API（logs/exec/attach）。典型建 Pod 流程 = 写 etcd → scheduler 绑定 → kubelet 起容器 → 回写状态（详见 [[concepts/控制平面与控制循环]]）。
- **端口号表**（Master/Worker）与**版本支持策略**：社区维护最新 3 个小版本，各 1 年补丁期；HA 中 apiserver 实例最多差 1 个小版本，kubelet 最多落后 apiserver 2 个小版本；升级顺序 apiserver 优先。

## 勘误 / 时效性

- ⚠️ 端口表过期：`8080` 明文端口（`--insecure-port`）已 v1.20 移除；kube-scheduler `10251`、kube-controller-manager `10252` 明文 healthz 已约 v1.23 移除（改 10259/10257 安全端口）；Kubelet cAdvisor `4194` 端口已移除。
- ⚠️ 文中"默认 DNS 是 kube-dns"已过期——**v1.13 起默认 CoreDNS**（见 [[entities/CoreDNS]]）。
- ⚠️ 版本示例（1.19–1.21）已远旧于当前发布线；版本表请以官方 Version Skew Policy 为准。

## 关联

[[concepts/集群架构]] · [[concepts/分层架构]] · [[entities/Kubernetes]]
