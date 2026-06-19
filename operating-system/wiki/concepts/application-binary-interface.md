---
title: "应用二进制接口（ABI）"
date: 2026-06-19
tags: [abi, 二进制接口, 调用约定, 系统调用, 可移植性, 指针宽度]
sources: ["core-dumped-operating-sytems-theory/Why Applications Are Operating-System Specific.en.srt"]
---

# 应用二进制接口（ABI）

**ABI（Application Binary Interface，应用二进制接口）**是一整套**底层二进制约定**，规定二进制
代码的各组件在**特定操作系统 + 架构**上如何交互。

> **API vs ABI**：正如 **API** 在**源码/应用层**定义"有哪些函数、参数是什么"，**ABI** 在
> **二进制层**定义"这些调用在机器码层面如何对接"——寄存器怎么用、参数放哪、数据如何布局等。
> API 让源代码可移植；ABI 决定**编译后的二进制**能否在某 OS/架构上正确运行。

## ABI 包含哪些约定

源视频以**系统调用**为主线举例（机制细节见 [[concepts/syscall-abi]]）：

- **系统调用号放在哪个寄存器**——两系统若用不同寄存器放调用号，机器码会读错寄存器、认错调用。
- **参数如何传递**——放寄存器、写内存、还是两者混用，因 OS 而异。例：Linux 上系统调用参数
  ≤6 个放寄存器，>6 个其余写入内存。
- **内存传参的方式**——用栈，还是某个专门的内存位置。
- **指针的地址宽度**——参数含指针时，32 位 vs 64 位必须一致。

> **（以下为通用/形式化知识，超出源视频范围）** ABI 不止于系统调用，正式的平台 ABI 规范还涵盖
> 函数调用约定（参数/返回值/寄存器保存规则）、数据类型大小与对齐、栈帧布局、
> [[concepts/executable-file-format]] 等。源视频只从**系统调用**角度引入这一概念，并把可执行文件
> 格式作为**另一层**单独讨论。

## 为何与可移植性相关

即便两个操作系统提供**完全相同的系统调用集合**，只要 ABI 有一处不对齐，针对一个 OS 编译的程序
在另一个 OS 上就可能**勉强执行却行为不可预测**（例如 OS 期望参数在内存、程序却放在寄存器）。
这是 [[analysis/why-applications-are-os-specific]] 中"第二层"不兼容的根源。

## 相关

- [[concepts/syscall-abi]] —— 系统调用 ABI 的具体机制（调用号/寄存器/系统调用表/指针校验）
- [[concepts/system-calls]] —— ABI 要对接的服务接口
- [[concepts/executable-file-format]] —— ABI 的一部分：可执行文件的二进制布局约定
- [[analysis/why-applications-are-os-specific]] —— ABI 作为 OS 绑定的第二层原因
- [[sources/why-apps-are-os-specific]] —— 来源
