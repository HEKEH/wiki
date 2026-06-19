---
title: "系统调用"
date: 2026-06-16
tags: [系统调用, syscall, sysret, 硬件抽象, 陷阱]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt"]
---

# 系统调用

**系统调用（system calls）**是操作系统以库的形式提供的函数，让用户程序能够访问那些在用户态
下无法直接触及的硬件资源（例如打开/读/写/关闭文件）。

## 机制

系统调用函数看起来像编译进用户程序的普通库代码，但用户程序运行在用户态，无法执行特权指令。
窍门在于：函数内部有一条指令（例如 `int`）触发一次软件 [[concepts/interrupts]]。该中断会：

1. 把程序计数器跳进操作系统的地址空间。
2. 把 [[concepts/mode-bit-and-operational-modes]] 翻转为内核态。

随后操作系统的例程执行完成该调用所需的特权指令。完成后，控制权交还调用者，模式位重置为 0
（用户态）——具体通过中断返回指令、专门的模式切换指令，或显式覆写模式位实现，视架构而定。

> 思维模型：函数的一部分运行在用户程序的地址空间（用户态），另一部分运行在操作系统的地址
> 空间（内核态）。

一些架构提供专门的 `syscall` / `sysret` 指令，省去手动设置中断，并让汇编更易读。

## 优点

- **硬件抽象**——用代码思考，而非用晶体管思考。
- **安全性**——操作系统中介所有硬件访问，即使糟糕的 C 程序也无法在物理上损坏硬件。
- **可移植性**——遵循共同规范的平台之间，同一份源代码可编译运行。

## 缺点

- **性能开销**——中断需要上下文切换，且与缓存配合不佳；操作系统拿到控制权后，可能先调度
  其他工作，因此不保证立即处理该调用。系统调用可被视为**陷阱（trap）**：抽象是诱饵，引诱
  程序交出控制权。
- **平台依赖性**——**每个操作系统提供自己独有的一套系统调用**（名字与功能都可能不同，如 Unix
  `fork`+`exec` vs Windows `CreateProcess`），加之 ABI 与可执行文件格式差异，导致应用程序不仅
  与架构绑定、也**与操作系统绑定**（见 [[analysis/why-applications-are-os-specific]]）。

## 相关

- [[concepts/syscall-abi]] —— 内核如何得知要执行哪个操作（调用号/传参/分发/指针校验）
- [[concepts/application-binary-interface]] —— 系统调用之下更广的二进制约定层
- [[analysis/why-applications-are-os-specific]] —— 系统调用差异如何让应用与 OS 绑定
- [[concepts/interprocess-communication]] —— 两种 IPC 模型都依赖系统调用进入内核
- [[concepts/interrupts]]
- [[concepts/mode-bit-and-operational-modes]]
- [[sources/single-bit-mode-bit-os-integrity]] —— 来源
