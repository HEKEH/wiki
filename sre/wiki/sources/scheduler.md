---
title: 源文档 - kube-scheduler
date: 2026-06-27
tags: [kubernetes, scheduler, 调度, 亲和性, taint, 调度框架, DRA, source]
sources: [kubernetes/components/scheduler.md]
---

# 源文档摘要：kube-scheduler

详述调度策略、亲和性、Taints、优先级、调度框架与较新特性。回答了 home.md "调度预选/优选细节"开放问题。

## 关键内容

- **两阶段**：预选（predicate/filter 过滤）+ 优选（priority/score 打分），附完整 predicates/priorities 算法清单。
- **约束位置**：nodeSelector、nodeAffinity（硬/软）、podAffinity/AntiAffinity（拓扑域 + topologyKey）。
- **Taints/Tolerations**：NoSchedule/PreferNoSchedule/NoExecute（含 tolerationSeconds）——节点驱逐的底层机制。
- **优先级与抢占**：PriorityClass（v1.11+ 默认开启）。
- **调度框架**（v1.19+）：扩展点 + 插件 + `KubeSchedulerConfiguration` profile。
- **较新特性**：DRA（v1.26+，GPU 等设备调度）、存储容量评分（v1.33 Alpha）。
- 多调度器（`schedulerName`）、MemoryPressure/DiskPressure 影响、Critical Pods。

## 勘误 / 时效性

- ⚠️ predicate/priority 是**经典调度器**词汇；自 v1.19 起官方实现已是调度框架插件（Filter/Score 扩展点）。`--policy-config-file` 仅 v1.23 前支持。
- DRA、存储容量评分为较新内容（v1.26/v1.33），方向准确。

## 关联

详见实体页 [[entities/kube-scheduler]] · [[concepts/控制平面与控制循环]] · [[concepts/Label与Selector]]
