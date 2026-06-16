# 操作系统 — 活动日志

按时间顺序、仅追加的 wiki 活动记录。

## [2026-06-16] ingest | Core Dumped —《处理器中的一个比特如何守护操作系统的完整性》

将原始 `.srt` 文字稿清洗为 [[sources/single-bit-mode-bit-os-integrity]]：删除了时间戳、
NeetCode 广告口播和口水话；修正了语音识别错误（NeetCode、`syscall`/`sysret`、Tencent、
MMU）。原始来源保持不变。这是本知识库的首次 ingest——在 CLAUDE.md 中设定了领域和类别。

将来源拆解为 6 个概念页——[[concepts/mode-bit-and-operational-modes]]、
[[concepts/interrupts]]、[[concepts/system-calls]]、[[concepts/memory-management-unit]]、
[[concepts/preemptive-vs-cooperative-os]]、[[concepts/kernel-mode-drivers]]——以及 3 个
实体页——[[entities/crowdstrike]]、[[entities/vanguard]]、[[entities/tencent]]。交叉链接了
所有页面，并更新了 index.md 与 home.md。

## [2026-06-16] lint | 全库改写为中文

将所有 wiki 页面（来源、概念、实体、home、index、log）从英文改写为中文，保留文件名/
wikilink 别名不变。技术术语保留英文（mode bit、MMU、`syscall`/`sysret`、BSOD 等）并附中文。

## [2026-06-16] query | 操作系统如何检测内核态代码破坏关键数据？

回答后归档为新分析页 [[analysis/detecting-kernel-corruption]]：核心结论是同级别基本无法
可靠检测（MMU 只能拦非法访问、拦不住对合法地址的覆盖），并梳理了页保护/W^X/SMEP-SMAP、
KASAN/Driver Verifier 红区、PatchGuard、VBS/HVCI 等补救机制。页面标注了源材料 vs 外部
工程知识。从 [[concepts/kernel-mode-drivers]] 与 [[concepts/memory-management-unit]] 回链，
并更新 index.md。

## [2026-06-16] query | 什么是缺页异常，如何检测到缺页异常

回答后归档为新概念页 [[concepts/page-fault]]：缺页异常是 MMU 在地址翻译时触发的同步异常
（present 位/权限位检查），经 IDT 进入内核处理；区分良性（按需调页/COW/栈增长）与非法
（SIGSEGV / panic / 蓝屏）。从 [[concepts/memory-management-unit]] 与
[[analysis/detecting-kernel-corruption]] 回链，并更新 index.md。页面标注了源材料 vs 外部
工程知识。

## [2026-06-16] query | 用户程序中断后，内核如何知道要执行哪个操作（如写硬盘）

回答后归档为新概念页 [[concepts/syscall-abi]]：系统调用号 + 寄存器传参（x86-64 ABI）+ 以调用号
查 `sys_call_table` 函数指针表分发 + `copy_from_user` 对用户指针做边界校验。补全了
[[concepts/system-calls]] 缺失的"内核如何得知意图"一环；从 system-calls 回链，并更新 index.md。
页面标注了源材料 vs 外部工程知识。
