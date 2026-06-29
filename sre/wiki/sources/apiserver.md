---
title: 源文档 - kube-apiserver
date: 2026-06-27
tags: [kubernetes, apiserver, REST-API, 访问控制, source]
sources: [kubernetes/components/apiserver.md]
---

# 源文档摘要：kube-apiserver

详述 apiserver 的 REST API、访问控制与较新的流式列表特性。

## 关键内容

- **两职责**：提供集群管理 REST API；作为数据中枢，是唯一直接读写 etcd 的组件。
- **REST API**：https（6443）安全接口；查询用 `kubectl api-versions`/`api-resources`，`--v=8` 看底层调用；可经 OpenAPI 生成各语言 client。
- **访问控制三阶段**：认证（401）→ 授权（403）→ 准入控制（仅写操作、多插件依次全过）。K8s 不直接管理用户。
- **流式列表响应（v1.33+）**：逐项序列化传输 List，内存峰值约 70–80GB→3GB（~20×），降低大集群 OOM。
- 启动参数示例、`kubectl proxy`/curl + ServiceAccount token 访问方式。

## 勘误 / 时效性

- ⚠️ `--insecure-port=8080` 明文 API **已 v1.20 移除**；`--enable-swagger-ui`、`/swaggerapi` 亦已废弃。
- ⚠️ `kubectl api-versions` 示例输出含大量 `*beta1`（如 `extensions/v1beta1`、`apps/v1beta1`），多数已在新版本毕业为稳定版或移除。
- 流式列表（v1.33）为较新且准确的内容。

## 关联

详见实体页 [[entities/kube-apiserver]] · [[concepts/控制平面与控制循环]] · [[concepts/插件机制与可扩展性]]
