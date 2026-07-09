---
title: CronJob
date: 2026-06-30
tags: [kubernetes, cronjob, 定时任务, cron, batch]
sources: [kubernetes/objects/cronjob.md]
---

# CronJob

CronJob 是**定时任务**控制器，类似 Linux 的 crontab：按时间表周期性地**创建 [[entities/Job]]**。关系链：**CronJob →（按 cron 表达式）创建 Job → 创建 [[entities/Pod]]**。典型场景：每天备份、每小时同步、定时清理/报表。

## 关键字段

| 字段 | 作用 |
| --- | --- |
| `.spec.schedule` | cron 表达式（如 `*/1 * * * *` 每分钟） |
| `.spec.jobTemplate` | 要运行的 Job 模板（格式同 [[entities/Job]]） |
| `.spec.startingDeadlineSeconds` | 错过调度点后仍可开始的截止期限（超过则跳过本次） |
| `.spec.concurrencyPolicy` | 并发策略，见下 |

## concurrencyPolicy（上次还没跑完，又到点了怎么办）

- **Allow**（默认）：允许并发，新旧 Job 同时跑。
- **Forbid**：禁止并发，跳过本次（等旧的结束）。
- **Replace**：用新 Job 替换尚未结束的旧 Job。

> 删除 CronJob 会一并删除它创建的 Job 和 Pod，并停止正在创建的 Job。

## API 版本

v1.5–v1.7 `batch/v2alpha1`（默认关）→ v1.8–v1.20 `batch/v1beta1` → **v1.21+ `batch/v1`（GA）**。

> ⚠️ 勘误：源示例 `kubectl run hello --schedule=...` 创建 CronJob **已不支持**（`kubectl run` 的 generator 早已移除，现只创建单个 Pod）——改用 `kubectl create cronjob` 或 YAML。

## 相关

- 被创建的一次性任务：[[entities/Job]]
- 控制器选型：[[concepts/工作负载控制器]]
