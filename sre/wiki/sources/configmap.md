---
title: ConfigMap（资源对象）
date: 2026-06-30
tags: [kubernetes, configmap, 配置, 配置分离]
sources: [kubernetes/objects/configmap.md]
---

# 源摘要：ConfigMap

feisky handbook `/concepts/objects/configmap.md` 的要点（深入页见 [[entities/ConfigMap]]）。

## 核心

- 保存**非敏感**配置的 key-value（单值或整份配置文件），实现**应用与配置分离**——改配置不必重建镜像。与 [[entities/Secret]] 类似但不加密。
- 创建：`--from-literal` / `--from-env-file` / `--from-file`（目录）/ YAML。

## 三种使用方式

1. **环境变量**：`env.valueFrom.configMapKeyRef`（单 key）或 `envFrom.configMapRef`（整份）。
2. **命令行参数**：先注入 env，再 `$(VAR_NAME)` 引用。
3. **Volume 挂载**：每个 key 成一个文件（key=文件名、value=内容）；`items` 选特定 key、改路径；`subPath` 挂单文件而不覆盖整个目录。

## 注意

- 必须先于引用它的 Pod 创建；Pod 只能用**同 namespace** 的 ConfigMap。
- **不可变 ConfigMap**（`immutable: true`，v1.21 GA）：禁改、关闭其 watch，降 apiserver 压力、防误更新。
