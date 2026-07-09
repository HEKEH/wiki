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

## [2026-06-27] ingest | Kubernetes 核心组件（components 目录，12 篇）

- 按指定顺序 ingest `raw/kubernetes/components/` 下 12 篇：components / apiserver / controller-manager / scheduler / kubelet / kube-proxy / kube-dns / etcd / federation / kubectl / kubeadm / hyperkube。
- 新建实体页：[[entities/kube-apiserver]]、[[entities/kube-controller-manager]]、[[entities/kube-scheduler]]、[[entities/kube-proxy]]、[[entities/CoreDNS]]、[[entities/kubeadm]]。
- 深化既有实体页：[[entities/etcd]]（Raft/v2v3/Watch/读一致性/对比 ZK·Consul）、[[entities/kubelet]]（pause 容器/CRI 客户端服务端/驱逐信号）、[[entities/kubectl]]（运维调试命令）、[[entities/Service]]（指向 kube-proxy/CoreDNS）。
- 概念页补充：[[concepts/控制平面与控制循环]]（新增"各组件深入"导航）。
- 新增 12 篇源摘要（sources/components…hyperkube），每篇含勘误/时效性标注。
- 解决开放问题：调度预选/优选细节（→ kube-scheduler）、reconcile/Informer（→ controller-manager）、Service iptables/IPVS 转发（→ kube-proxy）。
- 勘误汇总（源文档时效性）：8080 明文端口 v1.20 移除、10251/10252 healthz 与 cAdvisor 4194 端口移除、dockershim v1.24 移除、hyperkube v1.17 移除、Federation v1 废弃（→ Karmada）、Endpoints→EndpointSlices、Endpoint 选主锁→Lease、kube-dns→CoreDNS（v1.13 默认）、nftables 实为 v1.31 beta、policy-config 仅 v1.23 前。

## [2026-06-30] ingest | 资源对象之工作负载（objects 目录，第一批 7 篇）

- 从 feisky handbook `/concepts/objects` 下载全部 24 篇至 `raw/kubernetes/objects/`（经 `gh api` 取仓库原文，含 gateway-api、customresourcedefinition）；分批 ingest，本批为**工作负载** 7 篇：pod / replicaset / deployment / statefulset / daemonset / job / cronjob。
- 新建实体页：[[entities/ReplicaSet]]、[[entities/StatefulSet]]、[[entities/DaemonSet]]、[[entities/Job]]、[[entities/CronJob]]。
- 深化既有实体页：[[entities/Pod]]（生命周期/restartPolicy、init+原生 sidecar v1.33 GA、多容器模式、imagePullPolicy、优雅终止、PDB、原地资源调整）、[[entities/Deployment]]（RollingUpdate/Recreate、比例扩容、暂停恢复、状态 conditions、revisionHistoryLimit）。
- 概念页补充：[[concepts/工作负载控制器]]（表格与各节链到新实体页 + Job/StatefulSet/DaemonSet 进阶指针）、[[concepts/扩缩容与滚动升级]]（Recreate/暂停/历史指针）。
- 新增 7 篇源摘要（sources/pod…cronjob），每篇含勘误/时效性标注。
- 解决开放问题：StatefulSet 高级特性（有序部署、headless Service、volumeClaimTemplates、Partitions）→ [[entities/StatefulSet]]。
- 勘误汇总：StatefulSet/DaemonSet 默认更新策略实为 **RollingUpdate**（源称 OnDelete 已过时）；Pod "仅支持 Docker 镜像"过时（CRI/任意 OCI）；PriorityClass `scheduling.k8s.io/v1alpha1`→v1 GA；`extensions/v1beta1`/`apps/v1beta1`→apps/v1；Deployment `--record` 废弃、默认 maxSurge/maxUnavailable 25%、revisionHistoryLimit 默认 10；`kubectl run --schedule` 创建 CronJob 已移除；`kubectl get --show-all` 移除；DaemonSet 示例 `master` taint→`control-plane`；k8s.gcr.io→registry.k8s.io。
- 待续批次：网络（service/ingress/network-policy/gateway-api）、配置存储（configmap/secret/volume/pv/local-volume）、集群治理（namespace/node/quota/serviceaccount/security-context/podpreset/autoscaling/crd）。

## [2026-06-30] ingest | 资源对象之网络（objects 目录，第二批 4 篇）

- ingest service / ingress / network-policy / gateway-api。
- 新建实体页：[[entities/Ingress]]、[[entities/GatewayAPI]]、[[entities/NetworkPolicy]]。
- 深化 [[entities/Service]]：补齐四类型（含 LoadBalancer/ExternalName）、headless、无 selector 外部服务、源 IP 与 externalTrafficPolicy/internalTrafficPolicy、EndpointSlices（Endpoints v1.33 弃用）、多 Service CIDR、暴露层次（L4→L7）。
- 交叉引用：[[concepts/Pod网络模型]]、[[entities/Namespace]] 的 NetworkPolicy 由纯文本改为 wikilink。
- 新增 4 篇源摘要（sources/service/ingress/network-policy/gateway-api）。
- 解决开放问题：NetworkPolicy 细节 → [[entities/NetworkPolicy]]。
- 勘误汇总：Endpoints API v1.33 弃用→EndpointSlices；Ingress `extensions/v1beta1`+`serviceName/servicePort`→`networking.k8s.io/v1`+`backend.service.name`/`pathType`；Service 示例 kube-dns→CoreDNS；NetworkPolicy 示例 `protocol: tcp`→`TCP`、`kubectl run --replicas/--labels` 旧 generator 已变。
- 待续批次：配置存储（configmap/secret/volume/pv/local-volume）、集群治理（namespace/node/quota/serviceaccount/security-context/podpreset/autoscaling/crd）。

## [2026-06-30] ingest | 资源对象之配置与存储（objects 目录，第三批 5 篇）

- ingest configmap / secret / volume / persistent-volume / local-volume。
- 新建实体页：[[entities/ConfigMap]]、[[entities/Secret]]、[[entities/PersistentVolume]]（PV/PVC/StorageClass，并入 Local Volume）。
- 深化 [[concepts/Volume存储]]：补 image 卷、临时 vs 持久速览、subPath/Projected，PV/PVC/StorageClass 深机制指针至实体页，in-tree→CSI 勘误。
- 新增 5 篇源摘要（sources/configmap/secret/volume/persistent-volume/local-volume）。
- 勘误汇总：ServiceAccount 自动 token Secret v1.24 起取消→TokenRequest 投射卷；`--experimental-encryption-provider-config`→`--encryption-provider-config`；Recycle 回收策略弃用；in-tree 云存储插件→CSI；gitRepo 移除、FlexVolume 弃用；`storage.kubernetes.io/overlay·scratch`→`ephemeral-storage`；Local Volume v1.14 GA；新增 RWOP 访问模式。
- 待续批次：集群治理（namespace/node/quota/serviceaccount/security-context/podpreset/autoscaling/crd）。

## [2026-06-30] ingest | 资源对象之集群治理（objects 目录，第四批 8 篇，objects 全部完成）

- ingest namespace / node / quota / serviceaccount / security-context / podpreset / autoscaling / customresourcedefinition。
- 新建实体页：[[entities/ResourceQuota]]（含 LimitRange）、[[entities/ServiceAccount]]、[[entities/SecurityContext]]、[[entities/Autoscaling]]（HPA/VPA）、[[entities/CustomResourceDefinition]]。
- 深化既有实体页：[[entities/Node]]（自注册/Node Controller/Condition/cordon·drain/优雅·非优雅关闭）、[[entities/Namespace]]（内置 ns、级联删除、ResourceQuota 链接）。
- 概念页补充：[[concepts/扩缩容与滚动升级]]（指向 HPA/VPA 自动扩缩容）。
- 新增 8 篇源摘要；PodPreset 仅建源摘要（**v1.20 已移除**，不建实体页）。
- 解决开放问题：安全与访问控制（Secret/ServiceAccount/SecurityContext）→ 各自实体页。
- 勘误汇总：ServiceAccount 自动 token v1.24 取消→TokenRequest 投射卷、RBAC v1alpha1→v1；PodSecurityPolicy v1.21 弃用 v1.25 移除→PSA；PodPreset v1.20 移除；HPA `autoscaling/v2beta1`→`v2`(v1.23 GA)、Heapster 移除→metrics-server；CRD `apiextensions.k8s.io/v1beta1`→`v1`(v1.16 GA, v1beta1 v1.22 移除)、structural schema 强制；Node `rkt`/`OutOfDisk` 移除。
- **objects 目录 24 篇全部 ingest 完毕**（工作负载 7 + 网络 4 + 配置存储 5 + 治理 8）。后续可考虑：RBAC 单独概念页、Cluster Autoscaler、Karmada 多集群、版本支持周期权威来源。

## [2026-07-01] query | RBAC → 新建概念页

- 经多轮问答（RBAC 是什么、逐行拆解 Role/RoleBinding 示例），沉淀新概念页 [[concepts/RBAC]]。
- 内容：授权在 apiserver 三关卡中的定位、四对象 2×2（Role/ClusterRole × RoleBinding/ClusterRoleBinding）、三要素（subject/rule/默认拒绝白名单）、pod-reader 逐行示例、常见组合（含 ClusterRole+RoleBinding 复用）、内置 ClusterRole、与自定义控制器 RBAC 的关联。
- 交叉引用：[[entities/ServiceAccount]]、[[entities/CustomResourceDefinition]] 中"RBAC"纯文本改为 wikilink。
- 更新 `index.md`、`home.md`（把开放问题"RBAC 仍可单独建概念页"标记为已解答）。
