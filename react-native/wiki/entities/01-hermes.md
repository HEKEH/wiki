---
title: Hermes(JS 引擎)
date: 2026-07-09
tags: [hermes, js-engine, bytecode, performance]
sources: [2026-07-09-rn-advanced-guide-toc.md, 2026-07-09-rapidnative-performance-playbook.md]
---

# Hermes

Meta 为 React Native 打造的、随包分发的默认 JavaScript 引擎,自 RN 0.84 起为标准引擎。

## 要点

- **Bytecode 预编译**:构建期把 JS 编译为字节码(`.hbc`),省去运行时解析,显著改善**冷启动**。CI 中应核对 `.hbc` 确已随包分发。
- **Bundled Hermes**:引擎版本与 RN 版本绑定分发,避免版本错配(官方 architecture 有 Bundled Hermes 专页)。
- **Static Hermes**:更进一步的静态编译方向(见 RN Advanced Guide Ch4)。
- 配合 **Hermes Sampling Profiler** 定位 JS 线程瓶颈。→ [[concepts/04-performance-optimization]]

## 待深入

- Bundled Hermes 官方专页(`/architecture/bundled-hermes`)尚未抓取。
- Static Hermes 的适用场景与收益。

## 交叉引用

- 术语:[[concepts/05-glossary]]
- 来源:[[sources/05-rn-advanced-guide]]、[[sources/06-rapidnative-performance-playbook]]
