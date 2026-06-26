# SRE — Index

Catalog of all wiki pages, organized by category. Updated on every ingest.

## Overview

- [[home]] — 顶层综合与导航

## Entities

- [[entities/Kubernetes]] — 谷歌开源的容器集群管理系统（Borg 开源版），容器编排事实标准及其核心组件
- [[entities/etcd]] — 分布式强一致 KV 存储，K8s 唯一的集群状态数据库
- [[entities/Borg]] — 谷歌内部大规模集群管理系统，K8s 的前身与设计灵感来源
- [[entities/Pod]] — K8s 调度的基本单位，一组共享网络/IPC 的容器集合
- [[entities/Service]] — 应用服务抽象，按 label 提供 cluster IP、负载均衡与服务发现
- [[entities/Deployment]] — 经 ReplicaSet 管理 Pod 的控制器，负责扩缩容与滚动升级
- [[entities/Node]] — Pod 真正运行的主机（服务节点），运行 kubelet/runtime/kube-proxy
- [[entities/Namespace]] — 资源对象的抽象集合，用于逻辑隔离
- [[entities/kubectl]] — K8s 命令行工具，多数命令与 docker 对应
- [[entities/kubelet]] — 每个 Node 上的主 agent，确保 Pod 按期望运行并上报现状

## Concepts

- [[concepts/容器编排]] — 大规模容器化应用的自动化部署、调度、伸缩与自愈
- [[concepts/声明式API]] — 声明期望状态、由控制循环自动收敛的设计范式
- [[concepts/容器]] — OS 级虚拟化技术，用 namespace 隔离、镜像自包含
- [[concepts/Label与Selector]] — key/value 标签与选择器，松耦合关联资源的核心机制
- [[concepts/Volume存储]] — 持久化容器数据的卷抽象，广泛的存储后端支持（CSI）
- [[concepts/插件机制与可扩展性]] — CRI/CNI/CSI 等可插拔接口与功能扩展，平台化的基础
- [[concepts/扩缩容与滚动升级]] — Deployment 的副本伸缩与无中断升级、回滚
- [[concepts/健康检查]] — LivenessProbe / ReadinessProbe 探针
- [[concepts/资源限制]] — 经 cgroups 限制容器 CPU/内存
- [[concepts/集群架构]] — etcd + 控制节点 + 服务节点的集群组成与联邦
- [[concepts/控制平面与控制循环]] — apiserver/controller/scheduler/kubelet 的职责、协作与 reconcile loop
- [[concepts/Pod网络模型]] — 每 Pod 一 IP、CNI 分配、veth/Pod CIDR、端口冲突边界
- [[concepts/分层架构]] — 核心层/应用层/管理层/接口层/生态系统的类 Linux 分层
- [[concepts/设计理念]] — API/控制/架构/引导设计原则；容错性 + 易扩展性
- [[concepts/工作负载控制器]] — 业务类型→控制器：Deployment/Job/DaemonSet/StatefulSet（含 RC/RS）

## Sources

- [[sources/kubernetes简介]] — Kubernetes 简介（定位、组件、版本）
- [[sources/kubernetes基本概念]] — 核心对象与抽象（Container/Pod/Node/Service/Label 等）
- [[sources/kubernetes-101]] — 入门实操：kubectl、YAML、Volume、Service
- [[sources/kubernetes-201]] — 进阶：扩缩容、滚动升级、资源限制、健康检查
- [[sources/kubernetes集群]] — 集群组成、联邦、minikube / play-with-k8s
- [[sources/kubernetes架构原理]] — 架构原理：Borg、核心组件、Add-ons、分层架构（含本地图）
- [[sources/设计理念]] — 设计原则与核心 API 对象纵览（含勘误：StatefulSet/RBAC 已 GA 等）
