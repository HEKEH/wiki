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
- [[entities/ReplicaSet]] — 维持指定数量 Pod 副本的控制器，Deployment 的底层（含 RC 对比）
- [[entities/StatefulSet]] — 有状态应用控制器，提供稳定身份/存储/有序（headless svc + volumeClaimTemplates）
- [[entities/DaemonSet]] — 每节点一个 Pod 的守护控制器（日志/监控/系统程序）
- [[entities/Job]] — 一次性批处理控制器（并行模式、Indexed Job、successPolicy）
- [[entities/CronJob]] — 定时任务控制器，按 cron 周期创建 Job（并发策略 Allow/Forbid/Replace）
- [[entities/Node]] — Pod 真正运行的主机（服务节点），运行 kubelet/runtime/kube-proxy
- [[entities/Namespace]] — 资源对象的抽象集合，用于逻辑隔离
- [[entities/kubectl]] — K8s 命令行工具，多数命令与 docker 对应
- [[entities/kubelet]] — 每个 Node 上的主 agent，确保 Pod 按期望运行并上报现状
- [[entities/kube-apiserver]] — 控制平面唯一入口与数据中枢；REST API、认证/授权/准入、流式列表
- [[entities/kube-controller-manager]] — 集群大脑；各控制器、Leader Election、Informer、Node 驱逐
- [[entities/kube-scheduler]] — 调度器；预选/优选、亲和性、Taint、调度框架、DRA
- [[entities/kube-proxy]] — Service 数据面；userspace/iptables/ipvs/nftables 负载均衡
- [[entities/CoreDNS]] — 集群 DNS（v1.13 起默认，取代 kube-dns）；A/SRV 记录格式
- [[entities/kubeadm]] — 官方集群引导工具；static Pod 自举控制平面
- [[entities/Ingress]] — L7 路由规则集合，经 Ingress Controller 对外暴露 Service（host/path/TLS）
- [[entities/GatewayAPI]] — Ingress 下一代，面向角色、多协议（GatewayClass/Gateway/HTTPRoute）
- [[entities/NetworkPolicy]] — 基于 label 的网络隔离对象，默认全通、被选中即默认拒绝（依赖 CNI）
- [[entities/ConfigMap]] — 非敏感配置的 key-value，经 env/参数/卷注入，实现应用与配置分离
- [[entities/Secret]] — 敏感数据对象（Opaque/dockerconfigjson/sa-token），tmpfs 挂载、可静态加密
- [[entities/PersistentVolume]] — PV/PVC/StorageClass 持久存储抽象，动态供给与本地卷/扩容/拓扑
- [[entities/ResourceQuota]] — namespace 级资源配额与 LimitRange 默认值/区间，多租户治理
- [[entities/ServiceAccount]] — Pod 进程的机器身份（认证），配合 RBAC 授权；token v1.24 后投射卷
- [[entities/SecurityContext]] — 容器/Pod 安全上下文；capabilities/SELinux/用户命名空间，PSP→PSA
- [[entities/Autoscaling]] — HPA（自动调副本）/VPA（自动调资源），依赖 metrics-server
- [[entities/CustomResourceDefinition]] — 不改源码扩展 API 的 CRD，配控制器即 Operator

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
- [[concepts/RBAC]] — 基于角色的访问控制：Role/ClusterRole 定权限、RoleBinding 授主体（apiserver 授权关）

## Sources

- [[sources/kubernetes简介]] — Kubernetes 简介（定位、组件、版本）
- [[sources/kubernetes基本概念]] — 核心对象与抽象（Container/Pod/Node/Service/Label 等）
- [[sources/kubernetes-101]] — 入门实操：kubectl、YAML、Volume、Service
- [[sources/kubernetes-201]] — 进阶：扩缩容、滚动升级、资源限制、健康检查
- [[sources/kubernetes集群]] — 集群组成、联邦、minikube / play-with-k8s
- [[sources/kubernetes架构原理]] — 架构原理：Borg、核心组件、Add-ons、分层架构（含本地图）
- [[sources/设计理念]] — 设计原则与核心 API 对象纵览（含勘误：StatefulSet/RBAC 已 GA 等）
- [[sources/components]] — 核心组件总览：组件通信、端口表、版本支持策略（含端口/版本勘误）
- [[sources/apiserver]] — kube-apiserver：REST API、访问控制、流式列表（含 8080/beta 勘误）
- [[sources/controller-manager]] — controller-manager：控制器、选主、Informer、Node 驱逐
- [[sources/scheduler]] — scheduler：预选/优选、亲和性、Taint、调度框架、DRA
- [[sources/kubelet]] — kubelet：Pod 清单、static Pod、CRI、驱逐（含 dockershim/Heapster 勘误）
- [[sources/kube-proxy]] — kube-proxy：四种代理模式与 iptables/ipvs 实现
- [[sources/kube-dns]] — CoreDNS/kube-dns：DNS 记录格式与存根/上游 DNS
- [[sources/etcd]] — etcd 深入：Raft、v2/v3 存储与 Watch、对比 Zookeeper/Consul
- [[sources/federation]] — 集群联邦（Federation v1，**已废弃**，仅历史参考）
- [[sources/kubectl]] — kubectl 命令行全集（含 --generator/插件机制勘误）
- [[sources/kubeadm]] — kubeadm 部署流程（init/join、static Pod 引导）
- [[sources/hyperkube]] — hyperkube all-in-one binary（**v1.17 已移除**）
- [[sources/pod]] — Pod 资源对象：生命周期/restartPolicy/init+sidecar/PDB（含 Docker/优先级勘误）
- [[sources/deployment]] — Deployment：策略/比例扩容/暂停恢复/回滚历史（含 --record/apps-v1 勘误）
- [[sources/replicaset]] — ReplicaSet/RC：集合式 selector 与 legacy 说明
- [[sources/statefulset]] — StatefulSet：三种稳定、更新策略/Partitions（含 OnDelete 默认值勘误）
- [[sources/daemonset]] — DaemonSet：节点限定、更新回滚、静态 Pod 对比（含默认策略/master taint 勘误）
- [[sources/job]] — Job：并行模式、Indexed Job、backoffLimitPerIndex/successPolicy（v1.33）
- [[sources/cronjob]] — CronJob：schedule/concurrencyPolicy（含 kubectl run --schedule 勘误）
- [[sources/service]] — Service：四类型/headless/源 IP/EndpointSlices/多 CIDR（含 kube-dns/v1beta1 勘误）
- [[sources/ingress]] — Ingress：L7 路由/Controller/IngressClass/TLS（含 networking.k8s.io/v1 勘误）
- [[sources/network-policy]] — NetworkPolicy：默认全通、selector 隔离、依赖 CNI、不支持场景
- [[sources/gateway-api]] — Gateway API：角色分离、多协议、v1.3 特性（镜像/CORS/Inference）
- [[sources/configmap]] — ConfigMap：创建方式、三种使用、subPath、不可变
- [[sources/secret]] — Secret：三类型、tmpfs、静态加密、不可变（含 SA token v1.24 勘误）
- [[sources/volume]] — Volume：emptyDir/hostPath/projected/image、subPath、挂载传播（含 in-tree→CSI 勘误）
- [[sources/persistent-volume]] — PV/PVC/StorageClass：生命周期、accessModes、动态供给、v1.33 GA 特性
- [[sources/local-volume]] — LocalVolume：本地持久卷、静态 PV + nodeAffinity（含 GA v1.14 勘误）
- [[sources/namespace]] — Namespace：抽象集合、内置 ns、级联删除、操作
- [[sources/node]] — Node：自注册、Node Controller、Condition、cordon/drain、优雅关闭
- [[sources/quota]] — ResourceQuota/LimitRange：配额类型、scopes、与配额联动
- [[sources/serviceaccount]] — ServiceAccount：机器身份、RBAC、token（含 v1.24/v1alpha1 勘误）
- [[sources/security-context]] — SecurityContext：两级、supplementalGroupsPolicy、用户命名空间（含 PSP 移除）
- [[sources/autoscaling]] — HPA/VPA：指标驱动扩缩容（含 v2beta1/Heapster 勘误）
- [[sources/customresourcedefinition]] — CRD：扩展 API、validation/subresource/finalizer（含 v1beta1 移除）
- [[sources/podpreset]] — PodPreset（**已于 v1.20 移除**，仅历史；替代为 mutating webhook）
