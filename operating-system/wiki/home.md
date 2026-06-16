# Home

本知识库的顶层综合与导航。

## 关于

本 wiki 由 LLM 代理增量构建和维护。原始来源不可变；wiki 是 LLM 的图层——摘要、实体页、
概念页、交叉引用以及不断演进的综合。

## 快速链接

- [[index]] — 所有 wiki 页面的目录
- [[log]] — 按时间顺序的活动记录

## 当前状态

焦点：硬件与软件如何协作，让操作系统内核保持掌控。

核心洞见（来自 [[sources/single-bit-mode-bit-os-integrity]]）：CPU 中的一个**模式位**把
**内核态**（可执行特权指令）与**用户态**（不可执行）区分开。用户程序只能通过**中断**切入
内核态——这正是**系统调用**的工作方式（软件中断把控制权交给操作系统）。一个硬件**定时器
中断**让操作系统成为*抢占式*，使失控的循环无法饿死系统。驱动以及部分第三方软件（反作弊、
恶意软件检测）也运行在内核态，共享同一个地址空间——强大但危险（见 [[entities/crowdstrike]]、
[[entities/vanguard]]）。

**概念图：** [[concepts/mode-bit-and-operational-modes]] →
[[concepts/interrupts]] → [[concepts/system-calls]]；
[[concepts/memory-management-unit]]；[[concepts/preemptive-vs-cooperative-os]]；
[[concepts/kernel-mode-drivers]]。

## 开放问题

- 真实架构（x86 的 0–3 环、ARM 的异常等级）如何映射到这个简化的模式位模型？
- 有哪些现代缓解措施能缩小有 bug 的内核态代码的"爆炸半径"（例如 eBPF 沙箱、用户态驱动、
  Windows 在 CrowdStrike 事件后的改动）？
