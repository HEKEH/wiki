---
title: 源文档 - kubectl（命令行工具全集）
date: 2026-06-27
tags: [kubernetes, kubectl, 命令行, source]
sources: [kubernetes/components/kubectl.md]
---

# 源文档摘要：kubectl（命令行工具全集）

kubectl 各类子命令的参考大全。

## 关键内容

- **帮助/输出**：`-h`、`options`、`<cmd> --help`、`-o`（json/yaml/jsonpath/custom-columns）、`explain`。
- **配置**：cluster/user/context 三元组（`kubectl config`）。
- **CRUD**：run/create/get/set/patch/delete；`run` 资源类型由参数决定（默认 Deployment）。
- **运维/调试**：logs/attach/exec/cp/port-forward/proxy；drain/cordon/uncordon；auth can-i / reconcile；模拟用户（`--as`）；events 查询；插件（`~/.kube/plugins`、krew）；`get --raw` 原始 URI。

## 勘误 / 时效性

- ⚠️ `kubectl run` 的 `--generator` 参数已移除（现仅按 `--restart`/`--schedule` 决定类型，且 `run` 已主要面向创建 Pod）；`~/.kube/plugins` 老插件机制已被基于 `kubectl-` 前缀可执行文件 + krew 的新机制取代。
- 多数子命令（exec/logs/cp/drain/auth 等）仍准确通用。

## 关联

详见实体页 [[entities/kubectl]] · [[entities/kube-apiserver]] · [[concepts/扩缩容与滚动升级]]
