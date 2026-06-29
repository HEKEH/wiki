# Home

Top-level synthesis and orientation for this knowledge base.

## About

This wiki is incrementally built and maintained by an LLM agent. Raw sources are immutable; the wiki is the LLM's layer — summaries, entity pages, concept pages, cross-references, and evolving synthesis.

## Quick Links

- [[index]] — catalog of all wiki pages
- [[log]] — chronological activity record

## Current State

本知识库聚焦 **SRE / 站点可靠性工程**，当前内容已较系统地覆盖 **Kubernetes 入门**（概念 → 对象 → 集群 → 运维操作）。

核心洞见：

- [[entities/Kubernetes]] 源自 Google 的 [[entities/Borg]]，是容器编排的事实标准；它是一个**构建生态的平台**，而非封闭 PaaS。架构可从 [[concepts/分层架构]]（核心/应用/管理/接口/生态五层）理解。
- K8s 的可靠性根基在于 [[concepts/声明式API]]：用户声明期望状态，控制器循环自动收敛 —— 这"消除了编排的需要"（见 [[concepts/容器编排]]）。
- **对象层级**：[[concepts/容器]] → [[entities/Pod]]（调度单位）→ [[entities/Deployment]] → ReplicaSet 管理副本；[[entities/Service]] 经 [[concepts/Label与Selector]] 为 Pod 提供稳定入口。
- **集群结构**：[[entities/etcd]]（唯一状态源）+ 控制节点 + 服务节点（[[entities/Node]]），见 [[concepts/集群架构]]。
- **控制平面协作**：一切经 apiserver、各组件 watch 同一份状态接力（controller 决定"要什么"→ scheduler 选机器 → kubelet 真正跑），见 [[concepts/控制平面与控制循环]]。各组件现有深入页：[[entities/kube-apiserver]]（访问控制三关卡）、[[entities/kube-controller-manager]]（选主/Informer/Node 驱逐）、[[entities/kube-scheduler]]（预选优选/亲和性/Taint）、[[entities/etcd]]（Raft/v2v3/Watch）。
- **数据面与命名**：[[entities/kube-proxy]] 用 iptables/ipvs/nftables 把 Service 流量转发到 Pod；[[entities/CoreDNS]]（v1.13 起默认）提供集群 DNS。集群引导见 [[entities/kubeadm]]（static Pod 自举）。
- **平台化靠插件**：CRI/CNI/CSI 等可插拔接口（[[concepts/插件机制与可扩展性]]）让 K8s 适配一切环境而核心保持稳定。
- **运维能力**：[[concepts/扩缩容与滚动升级]]、[[concepts/健康检查]]、[[concepts/资源限制]] 共同支撑应用级自愈与可预测性能。
- **网络模型**：每个 Pod 一个独立 IP（network namespace + CNI 分配），同机 Pod 也各有 IP、端口互不冲突，见 [[concepts/Pod网络模型]]。
- **设计哲学**：两大核心理念是**容错性 + 易扩展性**；支撑原则包括声明式/幂等、控制逻辑只依赖当前状态、容错降级、可组合的名词式 API，见 [[concepts/设计理念]]。
- **工作负载分类**：按业务类型选控制器——无状态→Deployment、批处理→Job、每节点守护→DaemonSet、有状态→StatefulSet，见 [[concepts/工作负载控制器]]。

## Open Questions

- ~~K8s 控制器的 reconcile loop 具体如何实现？informer/缓存细节？~~ → 已解答，见 [[concepts/控制平面与控制循环]] 与 [[entities/kube-controller-manager]]（Informer = List 打底 + Watch 增量 + 本地只读缓存 + SharedInformer 共享 + 回调→workqueue→reconcile；etcd Watch→apiserver watch cache→informer 三层挡流量）。
- ~~调度器的调度策略（预选/优选）细节？~~ → 已解答，见 [[entities/kube-scheduler]]（predicate 过滤 + priority 打分的完整算法清单；现代为调度框架插件）。
- ~~有状态服务（StatefulSet）、PV/PVC 的完整模型？~~ → 已解答：StatefulSet 见 [[concepts/工作负载控制器]]，PV/PVC 见 [[concepts/Volume存储]]。（StatefulSet 高级特性如有序部署、headless Service、volumeClaimTemplates 待补）
- 安全与访问控制：Secret（默认仅 base64）、Service Account、RBAC —— 已在 [[sources/设计理念]] 登记，尚未单独建页。
- 网络模型细节：~~Pod 网络、CNI 如何分配 IP~~ → 已解答，见 [[concepts/Pod网络模型]]（每 Pod 一 IP、netns + veth + Pod CIDR）。~~Service 的 iptables/IPVS 转发实现~~ → 已解答，见 [[entities/kube-proxy]]（KUBE-SERVICES→SVC→SEP/DNAT 链路、ipvs+ipset）。NetworkPolicy 细节待补。
- 当前 K8s 版本支持周期与生态（源文档版本表已过期，待补充权威来源）。多个组件源文档存在时效问题（dockershim v1.24 移除、hyperkube v1.17 移除、Federation v1 废弃、Endpoints→EndpointSlices、明文端口移除等），已在各 source 页标注勘误。
- 多集群管理现状：源文档的 Federation v1 已废弃，现行方案（Karmada 等）待建页。
