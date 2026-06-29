---
title: 源文档 - kube-dns / CoreDNS
date: 2026-06-27
tags: [kubernetes, dns, coredns, kube-dns, source]
sources: [kubernetes/components/kube-dns.md]
---

# 源文档摘要：kube-dns / CoreDNS

详述集群 DNS：CoreDNS 取代 kube-dns、DNS 记录格式、存根/上游 DNS。

## 关键内容

- **CoreDNS**：v1.11 可用、**v1.13 起默认**，效率更高、资源更省，推荐替代 kube-dns。
- **DNS 记录**：Service A 记录 `<service>.<namespace>.svc.cluster.local`（普通→ClusterIP，Headless→Pod IP 列表）；SRV 记录用于命名端口；Pod A 记录 `<pod-ip>.<ns>.pod.cluster.local`。
- **存根域/上游 DNS**：经 ConfigMap 自定义（集群后缀→集群 DNS、存根域→私有 DNS、其余→上游）。
- **kube-dns 旧架构**：kube-dns（KubeDNS+SkyDNS）+ dnsmasq + sidecar 三容器。
- **常见问题**：Ubuntu systemd-resolved 写入 `127.0.0.53` 导致外网解析失败。

## 勘误 / 时效性

- "v1.13 默认 CoreDNS" 准确；kube-dns 三容器架构属历史细节（现以 CoreDNS 为主）。

## 关联

详见实体页 [[entities/CoreDNS]] · [[entities/Service]] · [[entities/Namespace]]
