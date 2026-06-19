---
title: "系统调用 ABI：内核如何知道要做什么"
date: 2026-06-16
tags: [系统调用, abi, 调用约定, 系统调用号, 系统调用表, copy_from_user, 安全]
sources: ["core-dumped-operating-sytems-theory/How a Single Bit Inside Your Processor Shields Your Operating System's Integrity.en.srt", "core-dumped-operating-sytems-theory/Why Applications Are Operating-System Specific.en.srt"]
---

# 系统调用 ABI：内核如何知道要做什么

> **范围说明**：第一篇源视频 [[sources/single-bit-mode-bit-os-integrity]] 只讲到"函数内部有条
> `int` 指令触发中断"。第二篇源视频 [[sources/why-apps-are-os-specific]] 补上了**总体机制**：
> 给每个系统调用配唯一编号、把编号写进寄存器、内核查系统调用表分发、以及参数经寄存器/内存
> 传递（Linux：≤6 个放寄存器）。本页更细的 Linux x86-64 具体约定（`rax`/`rdi…` 寄存器名、
> `copy_from_user` 指针校验）仍属源视频之外的工程知识，作为 [[concepts/system-calls]] 与
> [[concepts/application-binary-interface]] 的细节补充。以 Linux x86-64 为例。

中断只负责"敲门"切到内核态；真正的"我想要什么"是在敲门**之前**通过约定好的寄存器放好的。

## 1. 触发中断前：把"要做什么"写进约定寄存器

- **系统调用号**放进固定寄存器（x86-64 为 `rax`），它就是"指令编号"。如 `read`=0、`write`=1、
  `openat`=257。
- **参数**按序放进固定寄存器：`rdi, rsi, rdx, r10, r8, r9`。

这套"哪个寄存器放什么"的规矩就是 **ABI / 调用约定**，用户态与内核态**必须事先一致**——这正是
[[concepts/system-calls]] 里"函数一半在用户态、一半在内核态"能对接的原因：两半靠寄存器握手。

## 2. 示例：向硬盘写数据 `write(fd, buf, count)`

```
rax = 1            ; 系统调用号：write
rdi = fd           ; 写哪个文件（文件描述符）
rsi = buf          ; 数据在用户内存里的地址（指针）
rdx = count        ; 写多少字节
syscall            ; 敲门 → 进内核态
```

注意 `buf` 传的是**指针**而非数据本身。参数过多或数据量大放不下寄存器时，统一用"传一个指向
用户内存里结构体/缓冲区的指针"解决。

## 3. 进内核后：用调用号查表分发

1. 中断 / `syscall` 让 CPU 跳到内核**唯一入口**（陷阱/syscall 处理程序），并按 [[concepts/interrupts]]
   的机制保存寄存器、翻转模式位。
2. 内核从 `rax` 读出调用号，以它为**下标查系统调用表**（`sys_call_table`，本质是**函数指针
   数组**）→ 跳到对应内核函数（如 `sys_write`）。这一步完成"号码 → 具体操作"的映射。
3. 内核从保存的寄存器里取出参数（fd、指针、长度）。

## 4. 关键安全点：用户指针不能直接信

`buf` 是用户给的地址，内核**不能直接解引用**，而要用 `copy_from_user` 这类函数：**先校验该指针
确实落在该进程自己的地址空间内**，再把数据拷进内核。

- **内核为何能读用户内存**：它在内核态、控制着 [[concepts/memory-management-unit]]，可读写
  任何内存（含用户地址空间）。
- **为何必须校验**：否则用户程序传一个内核地址进来，就能骗内核替它读/写本不该碰的内存——经典
  提权漏洞。校验时若地址非法，会触发 [[concepts/page-fault]] 并被安全处理（返回 `-EFAULT`，
  而非崩溃）。

## 5. 返回

`sys_write` 把数据交给文件系统/块设备层，最终由**特权指令**操控磁盘控制器；写入字节数作为
**返回值放回 `rax`**，再 `sysret` 回用户态。出错则返回负的 errno。

## 一句话总结

中断负责"切到内核态"；寄存器里的"**调用号 + 参数**"负责"告诉内核**做什么、对谁做**"；内核用
调用号**查函数指针表分发**，并对所有用户指针做**边界校验**后才真正访问硬件。

## 相关

- [[concepts/system-calls]] —— 本页补全其"内核如何得知意图"的一环
- [[concepts/application-binary-interface]] —— 本页是 ABI 的系统调用具体实例；调用号/传参约定因 OS 而异
- [[analysis/why-applications-are-os-specific]] —— 这些约定不对齐如何导致应用与 OS 绑定
- [[concepts/interrupts]] —— 切到内核态、保存寄存器的机制
- [[concepts/memory-management-unit]] —— 内核可读用户内存、并据此校验用户指针
- [[concepts/page-fault]] —— 非法用户指针在校验时的兜底
- [[sources/single-bit-mode-bit-os-integrity]]、[[sources/why-apps-are-os-specific]] —— 来源
