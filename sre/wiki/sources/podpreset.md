---
title: PodPreset（资源对象，已移除）
date: 2026-06-30
tags: [kubernetes, podpreset, 已移除, 准入控制]
sources: [kubernetes/objects/podpreset.md]
---

# 源摘要：PodPreset（已移除）

feisky handbook `/concepts/objects/podpreset.md` 的要点。**该特性已不存在**，仅作历史记录，不单独建实体页。

## 它曾经是什么

- PodPreset 是一个**准入控制**对象：按 label selector **给匹配的 Pod 注入**额外的 env / envFrom / volume / volumeMount，免去在每个 Pod 模板里重复写（如统一注入数据库地址、时区卷）。
- Pod 加 annotation `podpreset.admission.kubernetes.io/exclude: "true"` 可豁免。
- API 为 `settings.k8s.io/v1alpha1`，默认不开启（需 `--runtime-config` + `PodPreset` 准入插件）。

## 勘误 / staleness

- ⚠️ **PodPreset 始终停留在 alpha，已于 v1.20 从 K8s 中移除**，`settings.k8s.io` API 组不复存在。
- 替代方案：**Mutating Admission Webhook**（如 Kyverno、OPA/Gatekeeper 的 mutate 规则）做注入，或直接把公共配置写进 Helm/Kustomize 模板。时区注入可用 Pod 的 `hostPath`/`spec.os`（见 [[entities/Pod]]）。
