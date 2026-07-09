---
title: Deployment（资源对象）
date: 2026-06-30
tags: [kubernetes, deployment, replicaset, 滚动升级, 回滚]
sources: [kubernetes/objects/deployment.md]
---

# 源摘要：Deployment

feisky handbook `/concepts/objects/deployment.md` 的要点（深入页见 [[entities/Deployment]]、[[concepts/扩缩容与滚动升级]]）。

## 核心

- Deployment 为 Pod 和 ReplicaSet 提供**声明式更新**，取代旧的 ReplicationController。
- 链路：**Deployment → ReplicaSet（名为 `<deploy>-<pod-template-hash>`）→ Pod**。
- 触发 rollout 的**唯一条件**：`.spec.template`（Pod 模板）变化（改镜像/label/env…）。扩缩容（改 `.spec.replicas`）**不触发** rollout、不产生 revision。

## 更新策略 `.spec.strategy.type`

- **RollingUpdate**（默认）：逐步新建/替换；`maxSurge`（最多超出期望数）+ `maxUnavailable`（最多不可用数）控制节奏（默认各 1，二者不可同时为 0）。
- **Recreate**：先杀光旧 Pod 再建新的（会中断）。

## 回滚与历史

- `kubectl rollout undo [--to-revision=N]`；revision 存于其控制的各 ReplicaSet。
- `.spec.revisionHistoryLimit` 限制保留的旧 RS 数；设 0 则**无法回滚**。
- `--record`（已废弃）记录 change-cause。

## 进阶行为

- **比例扩容（proportional scaling）**：rollout 进行中扩容时，新增副本按比例分摊到各 active RS，降低风险。
- **暂停/恢复**：`kubectl rollout pause/resume`——暂停期间改模板不触发 rollout，便于攒多次修改一次发布；暂停态不可回滚。
- **状态 conditions**：Progressing / Complete / Failed（`progressDeadlineSeconds` 超时 → `Reason=ProgressDeadlineExceeded`，但 K8s 不自动处理）。
- **Rollover**：rollout 进行中再次更新会立即转向新模板，不等旧 rollout 完成。
- Canary：源文档建议用多个 Deployment 实现（现代多用 Ingress/Gateway 或渐进式交付工具）。

## 勘误 / staleness

- ⚠️ 示例用 `--record` / `extensions/v1beta1`——`--record` 已废弃，Deployment 正式版为 **apps/v1**。
- 文中"未来 maxSurge/maxUnavailable 由 1-1 变 25%-25%""revisionHistoryLimit 未来默认 2"已成事实（现默认 25%/25%、保留 10 个 revision）。
