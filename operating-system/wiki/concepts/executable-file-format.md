---
title: "可执行文件格式（ELF / PE）"
date: 2026-06-19
tags: [可执行文件, ELF, PE, 文件格式, 加载器, 链接, 动态库, 元数据]
sources: ["core-dumped-operating-sytems-theory/Why Applications Are Operating-System Specific.en.srt"]
---

# 可执行文件格式（ELF / PE）

**可执行文件格式**是操作系统对"一个可执行文件内部如何组织"所定的规则。即便机器指令本身能被 CPU
执行，操作系统也得先**按自己认识的格式**把文件加载进内存——格式不对，加载这一步就过不去。

## 可执行文件里有什么

编译器翻译源代码后产出的可执行文件**不只含指令**，还包含：

- **指令**——CPU 要执行的机器码（text 段）。
- **数据**——初始化的全局/静态数据等。
- **元数据**——告诉操作系统：程序各部分(在这一堆 0/1 里)分别位于何处、如何加载与执行。

这套结构既保证可执行文件能被**正确加载运行**，也支撑**链接、调试、动态库**等特性。**每个操作
系统对此都有自己的规则**，因此同样的机器码装进不同格式的文件，另一个 OS 也认不出来。

## 两大代表

- **ELF**（Executable and Linkable Format）——Linux 使用。
- **PE**（Portable Executable）——Windows 使用。

> 注：PE 名字里的 "Portable" 指格式本身跨架构通用，**不**意味着 PE 程序能跨操作系统运行。

## 在 OS 绑定中的角色

这是 [[analysis/why-applications-are-os-specific]] 的"第三层"不兼容：即便系统调用与
[[concepts/application-binary-interface]] 都对齐，可执行文件格式不被目标 OS 识别，程序依然无法
被加载。

## 相关

- [[concepts/application-binary-interface]] —— 同属平台级二进制约定；源视频把二者列为不同层（形式化 ABI 规范常把文件格式一并规定）
- [[analysis/why-applications-are-os-specific]] —— 文件格式作为 OS 绑定的第三层原因
- [[sources/why-apps-are-os-specific]] —— 来源
