---
title: CronJob（资源对象）
date: 2026-06-30
tags: [kubernetes, cronjob, 定时任务, cron]
sources: [kubernetes/objects/cronjob.md]
---

# 源摘要：CronJob

feisky handbook `/concepts/objects/cronjob.md` 的要点（深入页见 [[entities/CronJob]]）。

## 核心

- CronJob = 定时任务，类似 Linux crontab：按 `.spec.schedule`（cron 表达式）周期性**创建 Job**。
- 关系链：**CronJob →（按 cron）创建 Job → 创建 [[entities/Pod]]**。

## 关键字段

- `.spec.schedule`：cron 表达式（如 `*/1 * * * *`）。
- `.spec.jobTemplate`：要运行的 Job 模板（格式同 [[entities/Job]]）。
- `.spec.startingDeadlineSeconds`：错过调度点后的开始截止期限。
- `.spec.concurrencyPolicy`：**Allow**（允许并发）/ **Forbid**（禁止，跳过本次）/ **Replace**（用新任务替换未结束的旧任务）。
- 删除 CronJob 会删除它创建的 Job 和 Pod。

## API 版本

- v1.5–v1.7 `batch/v2alpha1`（默认关）→ v1.8–v1.20 `batch/v1beta1` → **v1.21+ `batch/v1`（GA）**。

## 勘误 / staleness

- ⚠️ 源示例 `kubectl run hello --schedule=...` 创建 CronJob——**已不支持**（`kubectl run` 的 generator 早已移除，现仅创建单个 Pod）。改用 `kubectl create cronjob` 或 YAML。
