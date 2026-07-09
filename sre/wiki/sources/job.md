---
title: Job（资源对象）
date: 2026-06-30
tags: [kubernetes, job, 批处理, indexed-job, ttl]
sources: [kubernetes/objects/job.md]
---

# 源摘要：Job

feisky handbook `/concepts/objects/job.md` 的要点（深入页见 [[entities/Job]]）。

## 核心

- Job 负责**一次性批处理任务**，保证一个或多个 Pod **成功结束**。
- `restartPolicy` 仅支持 **OnFailure / Never**（不支持 Always）。
- **推荐用 Job 替代 Bare Pods**（裸 Pod 在 Node 重启后不会重建，Job 会）。

## 并行模式（`completions` × `parallelism`）

| 类型 | completions | parallelism |
| --- | --- | --- |
| 一次性 | 1 | 1 |
| 固定结束次数 | 2+ | 1 |
| 固定次数并行 | 2+ | 2+ |
| 工作队列并行 | 1 | 2+ |

## 关键字段

- `activeDeadlineSeconds`：失败重试的最大时间。
- `ttlSecondsAfterFinished`：**TTL 控制器**自动清理已结束（Complete/Failed）的 Job（要求各节点时间一致，如 NTP）。
- `.spec.suspend`（v1.21+ 稳定）：暂停/重启 Job。

## Indexed Job（v1.21+）

- `completionMode: Indexed`：给每个任务分配数值索引（经 annotation `batch.kubernetes.io/job-completion-index` 暴露）。
- **v1.33 稳定**：`backoffLimitPerIndex`（每索引独立重试预算）、`maxFailedIndexes`（允许失败索引上限）——适合"embarrassingly parallel"工作负载。
- **v1.33 GA**：`successPolicy`（按 `succeededCount`/`succeededIndexes` 定义成功，满足即提前终止其余 Pod）——适合 HPC / AI 训练 / 领导者-跟随者模式。

## 勘误 / staleness

- `kubectl get --show-all` 标志已移除（已结束 Pod 默认显示）。
