# SRE — Activity Log

Append-only chronological record of wiki activity.

<!-- Format: ## [YYYY-MM-DD] <operation> | <subject> -->
<!-- Operations: ingest, query, lint -->

## [2026-06-23] ingest | Kubernetes 简介

- 初始化 SRE 知识库：填写 `CLAUDE.md`（域、分类、约定），替换导航页占位符。
- 新建 [[sources/kubernetes简介]]、[[entities/Kubernetes]]、[[concepts/容器编排]]、[[concepts/声明式API]]。
- 更新 `index.md`、`home.md`。
- 备注：源文档版本支持表（v1.6–v1.11）已过期，已在源摘要中标注数据陈旧提示。

## [2026-06-23] ingest | Kubernetes 基本概念 / 101 / 201 / 集群

- 一次性 ingest 4 篇文章（对应 feisky.xyz introduction 的 concepts/101/201/cluster）。
- 新增源摘要：[[sources/kubernetes基本概念]]、[[sources/kubernetes-101]]、[[sources/kubernetes-201]]、[[sources/kubernetes集群]]。
- 新增实体页：[[entities/Pod]]、[[entities/Service]]、[[entities/Deployment]]、[[entities/Node]]、[[entities/Namespace]]、[[entities/kubectl]]。
- 新增概念页：[[concepts/容器]]、[[concepts/Label与Selector]]、[[concepts/Volume存储]]、[[concepts/插件机制与可扩展性]]、[[concepts/扩缩容与滚动升级]]、[[concepts/健康检查]]、[[concepts/资源限制]]、[[concepts/集群架构]]。
- 其中 [[concepts/Volume存储]] 与 [[concepts/插件机制与可扩展性]] 同时纳入了前几轮关于 Volume、插件、CRI/CNI/CSI 默认实现的问答内容。
- 更新 `index.md`、`home.md`。
- 备注：源文档中的 API 版本与命令偏旧（如 `extensions/v1beta1`、`kubectl rolling-update`），页面已标注其与 Deployment 新方式的区别，但未改写源文档。

## [2026-06-23] ingest | etcd（应答式补页）

- 应用户问题新建实体页 [[entities/etcd]]（分布式强一致 KV 存储 / Raft / Watch / 唯一状态源）。按 CLAUDE.md 分类归入 entities（etcd 是组件而非概念）。
- 内容超出源文档处（Raft、Watch、只有 apiserver 直连）为通用 K8s 知识补充。
- 把 [[concepts/集群架构]]、[[entities/Kubernetes]]、[[home]] 中 etcd 的纯文本提及替换为链接，更新 `index.md`。

## [2026-06-23] query | 控制平面协作 → 新建概念页

- 经多轮问答（apiserver↔controller、controller↔scheduler、"是否只是改 etcd"的澄清），沉淀新概念页 [[concepts/控制平面与控制循环]]。
- 内容：各组件职责表、"一切经 apiserver / watch apiserver" 原则、desired vs actual + reconcile loop、controller 与 scheduler 的接力关系、一次 scale 的端到端链路、"改 etcd 非目的、真实动作在 etcd 之外"的澄清与白板/工地类比。
- 更新 `index.md`、`home.md`（新增"控制平面协作"洞见，并把"reconcile loop 如何实现"开放问题标记为部分解答）。
- 内容多为通用 K8s 知识补充（源文档仅列组件清单）。

## [2026-06-23] query | Pod 网络模型 → 新建概念页

- 经多轮问答（跨 Pod 端口重复、同机多 IP），沉淀新概念页 [[concepts/Pod网络模型]]。
- 内容：每 Pod 一 IP 原则、network namespace、CNI 分配（Pod CIDR + veth pair）、端口冲突边界（containerPort 占 Pod IP 不冲突 vs hostPort/NodePort 占 Node IP 可能冲突）、访问与隔离、验证命令。
- 更新 `index.md`、`home.md`（新增"网络模型"洞见，并把"Pod 网络 / CNI 分配 IP"开放问题标记为已解答；剩 Service iptables/IPVS 与 NetworkPolicy 细节待补）。
- 期间另对 [[entities/Service]] 补充了 targetPort 数字 vs 命名端口、多端口；对 [[entities/Namespace]] 补充了隔离边界（默认不隔离网络 + NetworkPolicy）。

## [2026-06-26] ingest | Kubernetes 架构原理

- 下载源文档 `raw/kubernetes/kubernetes架构原理.md`（feisky.xyz/concepts/architecture）及 7 张配图到 `raw/kubernetes/images/`（按原始 /files/ ID 命名）。经用户确认，将 raw markdown 的图片引用改写为本地路径（破例修改 raw）。
- 新建 [[sources/kubernetes架构原理]]（含本地架构图/分层图）、[[entities/Borg]]（K8s 前身，含 Borg↔K8s 组件对应表）、[[concepts/分层架构]]（核心/应用/管理/接口/生态五层）。
- 更新 [[entities/Kubernetes]]（Borg/分层架构链接）、[[concepts/集群架构]]（Borg/分层架构链接 + 推荐 Add-ons 小节）、`index.md`、`home.md`。
- 数据陈旧提示：源文档"默认运行时 Docker"、Heapster、kube-dns、Federation"跨可用区"等已过时，已在源摘要与相关页标注。

## [2026-06-26] query | kubelet → 新建实体页

- 经多轮问答（kubelet ↔ container runtime、核心层是否使用 CRI/CNI/CSI、kubelet 是否等于核心层），沉淀新实体页 [[entities/kubelet]]。
- 内容：与分层架构的关系（kubelet ≠ 核心层，仅实现其执行环境一块）、内部模块表（syncLoop/PLEG/Volume Manager/Probe Manager/cAdvisor/cgroup/Eviction/Device Manager 等）、作为 CRI/CNI/CSI 调用方、静态 Pod 与自举、kubelet≠kubectl 提示。
- 把 [[entities/Node]]、[[concepts/控制平面与控制循环]] 中 kubelet 的纯文本提及链接化；更新 `index.md`。
- 顺手修复 [[concepts/控制平面与控制循环]] 两处代码块缺少语言标注（MD040）。

## [2026-06-26] ingest | 设计理念（含勘误）

- ingest `raw/kubernetes/设计理念.md`，按"只补充必要知识"原则：仅就新内容建页，不复制已有的 Pod/Service/Node/Namespace 等。
- 新建 [[concepts/设计理念]]（API/控制/架构/引导原则 + 容错性/易扩展性）、[[concepts/工作负载控制器]]（业务类型→控制器，含 RC/RS/Job/DaemonSet/StatefulSet，回答 StatefulSet 开放问题）、[[sources/设计理念]]。
- 补充：[[concepts/声明式API]] 加 metadata/spec/status 三段结构 + 幂等；[[concepts/Volume存储]] 加 PV/PVC（PV↔Node、PVC↔Pod 对称关系，回答 PV/PVC 开放问题）。
- **勘误/陈旧标注**（核心任务之一）：StatefulSet 自 1.9 GA（非源文档所说 Alpha）；RBAC 自 1.8 GA 且为默认（非 1.3 alpha）；RC 已 legacy；Federation v1 已废弃；Secret 默认仅 base64 非加密；默认命名空间现 4 个非 2 个。详见 [[sources/设计理念]]。
- 未建页（仅在源摘要登记）：Secret、User/Service Account、RBAC——按需后续 ingest。
- 更新 `index.md`、`home.md`（新增设计哲学/工作负载洞见，标记 StatefulSet、PV/PVC 开放问题为已解答）。
