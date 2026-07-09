---
title: CustomResourceDefinition（资源对象）
date: 2026-06-30
tags: [kubernetes, crd, 自定义资源, operator, kubebuilder]
sources: [kubernetes/objects/customresourcedefinition.md]
---

# 源摘要：CustomResourceDefinition

feisky handbook `/concepts/objects/customresourcedefinition.md` 的要点（深入页见 [[entities/CustomResourceDefinition]]）。

## 核心

- **CRD（v1.7+）**：无需改 K8s 源码即可**扩展 API**、注册自定义对象（取代已弃用的 TPR）。声明后即得 `/apis/<group>/<version>/<plural>` REST 端点，可用 kubectl 增删查改。
- `scope`：Namespaced 或 Cluster；`names`（plural/singular/kind/shortNames）；`versions`（每版本 served/storage）。

## 关键能力

- **Validation**：基于 OpenAPI v3 schema 校验字段（类型/范围/正则）；v1 要求 **structural schema**。
- **Subresources**：`/status` 与 `/scale`（让 `kubectl scale` 和 HPA 能作用于自定义资源）。
- **Finalizer**（`metadata.finalizers`）：删除时先置 `deletionTimestamp` 触发控制器清理，移除 finalizer 后才真正删——异步预删除钩子。
- **Categories**：分组以便 `kubectl get <category>`。

## CRD + 控制器 = Operator

- CRD 只定义"数据结构"，需配一个**控制器** watch 它并 reconcile（见 [[concepts/控制平面与控制循环]]）——这就是 **Operator 模式**。`kubebuilder`/sample-controller 提供脚手架。

## 勘误 / staleness

- ⚠️ API：`apiextensions.k8s.io/v1beta1` 已移除——现为 **`apiextensions.k8s.io/v1`（GA 自 v1.16，v1beta1 于 v1.22 删除）**；`version` 单数字段改 `versions` 列表；CustomResourceValidation/Subresources 等门控早已 GA、structural schema 成强制。
