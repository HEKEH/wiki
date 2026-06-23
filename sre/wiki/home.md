# Home

Top-level synthesis and orientation for this knowledge base.

## About

This wiki is incrementally built and maintained by an LLM agent. Raw sources are immutable; the wiki is the LLM's layer — summaries, entity pages, concept pages, cross-references, and evolving synthesis.

## Quick Links

- [[index]] — catalog of all wiki pages
- [[log]] — chronological activity record

## Current State

本知识库聚焦 **SRE / 站点可靠性工程**，当前内容从**容器编排**主题起步。

核心洞见：

- [[entities/Kubernetes]] 源自 Google 的 Borg，是容器编排的事实标准；它是一个**构建生态的平台**，而非封闭 PaaS。
- K8s 的可靠性根基在于 [[concepts/声明式API]]：用户声明期望状态，控制器循环自动收敛 —— 这"消除了编排的需要"（见 [[concepts/容器编排]]）。
- 控制平面（etcd / apiserver / controller manager / scheduler）与节点组件（kubelet / runtime / kube-proxy）分工明确，etcd 是唯一状态源。

## Open Questions

- K8s 控制器的 reconcile loop 具体如何实现？（informer/watch 机制）
- Pod、Service、Deployment 等核心资源对象的关系与生命周期？
- 调度器的调度策略（预选/优选）细节？
- 当前 K8s 版本支持周期与生态（源文档版本表已过期，待补充权威来源）。
