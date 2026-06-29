---
title: kube-apiserver
date: 2026-06-27
tags: [kubernetes, apiserver, 控制平面, REST-API, 访问控制]
sources: [kubernetes/components/apiserver.md, kubernetes/components/components.md]
---

# kube-apiserver

kube-apiserver 是 Kubernetes 控制平面**最核心**的组件，承担两个职责：

1. 对外/对内提供集群管理的 **REST API**（认证、授权、准入、数据校验、API 注册与发现）；
2. 充当所有组件间的**数据中枢**——它是**唯一直接读写 [[entities/etcd]] 的组件**，其余组件（controller-manager、scheduler、[[entities/kubelet]]、kube-proxy）都经它查询/修改数据（见 [[concepts/控制平面与控制循环]]）。

## REST API 接口

- **https（默认 6443）**：安全接口，所有请求需经访问控制。
- ~~http（127.0.0.1:8080）非安全接口~~：源文档描述的 `--insecure-port=8080` 已废弃——**v1.20 起只能设为 0（默认禁用），flag 于 v1.24 彻底移除**，现代集群只有安全端口。

常用查询：

```bash
kubectl api-versions       # 支持的 API group/version
kubectl api-resources      # 支持的资源对象
kubectl --v=8 get pods     # 打开调试日志，看到底层 REST 调用
kubectl get --raw /api/v1/namespaces   # 直接访问原始 API
```

访问方式：kubectl、各语言 client SDK（[client-go](https://github.com/kubernetes/client-go) 等）、`kubectl proxy` HTTP 代理、带 ServiceAccount token 的 curl。

## 访问控制：每个请求的三道关卡

API 请求依次经过三个阶段，全部通过才被接受（见 [[concepts/插件机制与可扩展性]] 的准入 webhook）：

| 阶段 | 作用 | 失败返回 | 特点 |
| --- | --- | --- | --- |
| **认证（Authentication）** | 确认"你是谁"，得到 username | HTTP 401 | 多插件，任一通过即可 |
| **授权（Authorization）** | 确认"你能否做这个操作" | HTTP 403 | 多插件（RBAC 等），任一通过即可 |
| **准入控制（Admission Control）** | 校验/修改请求**内容**，可加默认值 | 取决于插件 | 多插件**依次**调用，全通过才放行；仅作用于写操作（创建/更新/删除/连接），对读无效 |

> **K8s 不直接管理用户**：认证/授权用到 username，但 K8s 不存储 user 对象，用户身份由外部（证书、OIDC、ServiceAccount 等）提供。

**为何授权之外还要准入？** 授权只看 `(用户, 动词, 资源类型)`、内容无关、且只能 yes/no——管不了"能建 Pod 但不准建特权容器""镜像必须来自公司仓库""必须带 resource limits""配额已满则拒绝"这类**基于对象内容/集群状态**的策略，也**不能改写请求**。准入正好补这两块：① 按内容+状态校验（Pod Security、ResourceQuota、LimitRanger 等）；② **改写请求**（注入默认值/sidecar/SA token，授权永远做不到）。类比：授权=门禁刷卡（凭身份能否进楼），准入=进门安检（查带的东西）+ 发工牌（补默认装备）。

## 流式列表响应（Streaming List，v1.33+）

为解决大规模集群 List API 的内存峰值问题，v1.33 起 apiserver 支持流式编码：逐个序列化并传输 `Items`，内存可在传输中逐步释放。基准显示峰值内存从 70–80GB 降至 ~3GB（约 20×），降低 OOM 风险，且字节级兼容、客户端无需改动。适用于大集群（>1000 节点）的列表/监控/CI 状态检查。

- **旧做法**：把整个 List 一次性序列化成一大块连续内存再发，整份字节流同时驻留 → 内存 ≈ `并发数 × 整份响应大小`，易冲高 OOM。
- **新做法**：对 `Items` **逐项序列化→写出→释放**，任意时刻只占一个 item 的缓冲，边发边放。
- **为何无需改客户端**：发出的字节流与旧方式**逐字节相同**（仍是合法 List），纯服务端流式吐出。类比：旧=整摞复印好再递，新=复印一页递一页，内容一样但工作台峰值天差地别。
- 与分页（`limit`/`continue`）、watch cache **互补**——即便客户端要全量 List 也能压住峰值。

## 端口

默认安全端口 **6443**；etcd client API 2379-2380；Kubelet API 10250。

> ⚠️ 源文档的端口表已部分过期：apiserver 明文 8080 已禁用/移除（见上）；kube-scheduler 10251 / kube-controller-manager 10252 的 **healthz 明文端口约 v1.23 移除**（改为 10259 / 10257 安全端口）；Kubelet cAdvisor 4194 端口亦已移除。

## 相关

- 唯一的"守门人"为何重要：[[entities/etcd]]、[[concepts/集群架构]]
- 组件协作与端到端链路：[[concepts/控制平面与控制循环]]
- 命令行客户端：[[entities/kubectl]]
