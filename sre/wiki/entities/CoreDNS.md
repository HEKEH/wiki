---
title: CoreDNS / kube-dns
date: 2026-06-27
tags: [kubernetes, dns, coredns, kube-dns, 服务发现]
sources: [kubernetes/components/kube-dns.md]
---

# CoreDNS / kube-dns

DNS 是 Kubernetes 的核心扩展之一，为集群提供**命名服务**：Pod 内可用域名访问 [[entities/Service]]，是服务发现的基础（配合 [[entities/kube-proxy]] 的 ClusterIP 转发）。

> **是插件吗？** 两者都不是编进 apiserver/kubelet 的核心组件，而是以 Pod 形式部署在 `kube-system` 的**集群扩展组件（addon）**、可替换。区别在来源：**kube-dns** 由 K8s 项目自研；**CoreDNS** 是独立的 **CNCF 毕业项目**（K8s 之外也通用），被 K8s 采纳为默认。这属于 addon 机制，与 [[concepts/插件机制与可扩展性]] 的 CRI/CNI/CSI **接口插件不是一回事**（DNS 无标准接口）。另注：CoreDNS 自身的 "plugin" 链（`kubernetes`/`forward`/`cache` 等）是它**软件内部**的插件架构，与 K8s 插件机制无关。

## CoreDNS 取代 kube-dns

- **CoreDNS**：v1.11 起可用，**v1.13 起成为默认 DNS**。效率更高、资源占用更小、单进程插件化，推荐替代 kube-dns。
- **kube-dns**（旧）：单 Pod 三容器——`kube-dns`（KubeDNS 监听 Service/Endpoint 变化 + SkyDNS 解析）、`dnsmasq`（缓存）、`sidecar`（健康检查/metrics）。已被 CoreDNS 取代。

> 源文档"v1.13 起默认 CoreDNS"正确；但 [[sources/components]] 里"默认 DNS 是 kube-dns"的表述已过期。

## DNS 记录格式

**Service：**

- A 记录：`<service>.<namespace>.svc.cluster.local`
  - 普通 Service → 解析为 **ClusterIP**
  - Headless Service（`clusterIP: None`）→ 解析为**后端 Pod IP 列表**
- SRV 记录：`_<port-name>._<protocol>.<service>.<namespace>.svc.cluster.local`（用于命名端口发现）

> 名字顺序是 `<service>.<namespace>`（小范围在前、大范围在后），与 Namespace 隔离/DNS 已在 [[entities/Namespace]]、[[entities/Service]] 详述。

**Pod：**

- A 记录：`<pod-ip-with-dashes>.<namespace>.pod.cluster.local`（IP 中的 `.` 换成 `-`）
- 指定 `hostname` + `subdomain` 时：`<hostname>.<subdomain>.<namespace>.svc.cluster.local`

## 存根域与上游 DNS

可经 ConfigMap 自定义：集群后缀（`.cluster.local`）的查询发给集群 DNS，存根域（如 `.acme.local`）发给指定私有 DNS，其余发给上游 DNS（如 `8.8.8.8`）。

## 常见问题

Ubuntu 18.04+ 的 systemd-resolved 会把 `nameserver 127.0.0.53` 写入 `/etc/resolv.conf`（本地回环地址），导致 CoreDNS/kube-dns 无法解析外网。解决：让 DNS 用 `/run/systemd/resolve/resolv.conf`。

## 相关

- 服务抽象与命名端口：[[entities/Service]]
- 命名空间与 DNS 域：[[entities/Namespace]]
- 数据面转发：[[entities/kube-proxy]]
