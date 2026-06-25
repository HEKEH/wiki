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
