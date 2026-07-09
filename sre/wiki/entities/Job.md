---
title: Job
date: 2026-06-30
tags: [kubernetes, job, 批处理, indexed-job, ttl, hpc]
sources: [kubernetes/objects/job.md]
---

# Job

Job 控制**有头有尾的一次性批处理任务**：创建一个或多个 [[entities/Pod]]，保证**成功结束**指定数量后即完成。判断用 Job 还是 [[entities/Deployment]]：**这活儿有"做完"的概念吗？有→Job，没有（要常驻）→Deployment**（见 [[concepts/工作负载控制器]]）。

- `restartPolicy` 仅支持 **OnFailure / Never**（不支持 Always——常驻语义与"跑完即止"矛盾）。
- **推荐用 Job 替代 Bare Pods**：裸 Pod 在 Node 重启后不会重建，Job 会另起 Pod 续跑。
- Job Controller 监控 Pod，失败则按 `restartPolicy` 决定是否新建 Pod 重试。

## 并行模式（`completions` × `parallelism`）

| 类型 | 行为 | completions | parallelism |
| --- | --- | --- | --- |
| 一次性 | 一个 Pod 成功即完成 | 1 | 1 |
| 固定结束次数 | 依次跑到 N 个成功 | 2+ | 1 |
| 固定次数并行 | 多 Pod 并行跑到 N 个成功 | 2+ | 2+ |
| 工作队列并行 | 任一成功即算成功 | 1 | 2+ |

## 关键字段

- `activeDeadlineSeconds`：Job 的**硬性墙钟时限**。超时即**主动终止所有运行中 Pod**、把 Job 判 **Failed（`DeadlineExceeded`）**、不再重试——按**时间**判败（优先级高于按失败次数的 `backoffLimit`，到点即杀正在跑的 Pod）。计的是活跃运行时间，suspend 期间不计、恢复时重置。
- `ttlSecondsAfterFinished`：**TTL 控制器**在 Job 结束（Complete/Failed）后经过该秒数自动清理它（连带 Pod）。**不设=永久保留**（高频 Job 会堆积、拖累 apiserver/etcd）；**0=立即删**（无法查日志）；设 600 即留 10min 排查窗口后清理（要求各节点时间一致，如 NTP）。
- `.spec.suspend`（v1.21+ 稳定）：`true`=**暂停**（不建 Pod；若 Job 在跑则**删掉运行中 Pod**，非原地冻结），`false`=运行；conditions 记 `Suspended`，恢复时计时重置。典型用于**排队调度**（如 Kueue 先建挂起、等配额再放行）、延迟启动、临时叫停止损。

## Indexed Job（v1.21+）

`completionMode: Indexed` 给每个任务分配**数值索引**（经 annotation `batch.kubernetes.io/job-completion-index` 暴露，容器内读 `$JOB_COMPLETION_INDEX`），适合静态任务切分。配套两个 **v1.33 稳定**特性，专为"embarrassingly parallel"负载设计：

- **`backoffLimitPerIndex`**：每个索引**独立**的重试预算——单索引失败不再吃掉整个 Job 的失败预算。
- **`maxFailedIndexes`**：允许失败的索引数上限。

## 成功策略 successPolicy（v1.33 GA，仅 Indexed Job）

用 `rules` 列表定义"提前判成功"的条件；**一旦满足即终止其余运行中的 Pod**（加 `SuccessCriteriaMet` 条件），省算力。典型：科学仿真、AI/ML 训练、HPC、领导者-跟随者模式。

语义要点：

- **`rules` 之间是 OR**：满足任一条即判 Job 成功（多写几个 `-` 即多个"或"条件）。
- **一条 rule 内部**：只写 `succeededIndexes`（如 `"0-3"`）= 这些索引**全部**成功；只写 `succeededCount` = **任意**这么多个索引成功；**两者都写**（如 `succeededIndexes: "0-4"` + `succeededCount: 3`）= "**这些索引里至少 N 个成功**"（count 限定在该 subset 内，是 AND 式组合，故 `succeededCount` 须 ≤ 集合大小）。
- **跨 rule 无 AND**：`rules` 只有 OR；要"AND"只能靠单条 rule 的 count-within-subset 表达，独立条件的真 AND 不支持。

## 示例（尽量用全参数的 Indexed Job）

`backoffLimitPerIndex`、`maxFailedIndexes`、`successPolicy` 都**仅 Indexed 模式**可用，故用 Indexed Job 演示：

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: sim-batch, namespace: default }
spec:
  # 并行模式：固定次数并行
  completions: 10            # 要成功的索引数(0~9)
  parallelism: 3             # 最多 3 个 Pod 同时跑
  completionMode: Indexed    # 每 Pod 分一个数值索引 → $JOB_COMPLETION_INDEX
  # 重试预算(per-index 与全局 backoffLimit 二选一：这里用 per-index 一套)
  backoffLimitPerIndex: 3    # 每个索引独立重试 3 次(10 个索引各算，共最多 ~30 次尝试)
  maxFailedIndexes: 2        # 判 Job 失败的开关：失败索引超过 2 个即整体失败
                             # (设了 backoffLimitPerIndex 后，全局 backoffLimit 不再是判败依据，故此处不写)
  # 时间与清理
  activeDeadlineSeconds: 3600     # 硬时限：到点即杀掉所有运行中 Pod、Job 判 Failed(DeadlineExceeded)
  ttlSecondsAfterFinished: 600    # 结束后 10min 由 TTL 控制器自动清理
  suspend: false             # true=先挂起不起 Pod，可后期改 false 恢复
  # 成功策略(仅 Indexed)：满足任一 rule 即判成功、终止其余 Pod、加 SuccessCriteriaMet
  successPolicy:
    rules:                        # 多条 rule 之间是 OR
    - succeededIndexes: "0"       # 规则①：索引 0(领导者)成功
    - succeededCount: 8           # 规则②：或任意 8 个索引成功
  template:
    spec:
      restartPolicy: Never        # Job 仅支持 Never / OnFailure
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c"]
        args: ["echo 处理分片 $JOB_COMPLETION_INDEX / 共 $JOB_COMPLETIONS"]
```

> 搭配注意：`backoffLimitPerIndex`/`maxFailedIndexes`/`successPolicy` 均**要求 `completionMode: Indexed`**；容器内 `$JOB_COMPLETION_INDEX`=当前分片号、`$JOB_COMPLETIONS`=总数，据此切数据。

## 相关

- 定时触发 Job：[[entities/CronJob]]
- 控制器选型：[[concepts/工作负载控制器]]
