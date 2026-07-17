---
title: 官方文档 — Performance Overview
date: 2026-07-09
tags: [official-docs, performance, frame-rate, profiling]
sources: [2026-07-09-reactnative-dev-performance.md]
---

# 官方文档:Performance Overview

官方性能主线:确立 60fps / 16.67ms 帧预算与"JS 线程 vs UI 线程"双帧率心智模型,并列出最常见的性能坑与解法。综合实践见 [[concepts/04-performance-optimization]]。

## 核心要点

- 目标 60fps(每帧 16.67ms);分别监控 **JS frame rate** 与 **UI frame rate**(Perf Monitor)。
- JS 线程:业务/触摸/state;复杂重渲染会丢帧冻结交互。UI 线程:原生动画与滚动,更有韧性。
- 常见坑:dev 模式测性能、`console.log`、大列表 FlatList、动画期 JS 重活、透明度合成移动视图、动画图片尺寸、TouchableX 重渲染。
- 官方哲学:尽量自动优化,只有 RN 无法判断时才需手动介入。

## 链接的官方子指南(待抓取)

Profiling、Build speed、Optimizing FlatList Configuration、Optimizing JavaScript loading、Animations、Debugging。

## 交叉引用

- 系统方法:[[concepts/04-performance-optimization]]
- 线程模型:[[concepts/03-threading-model]]
