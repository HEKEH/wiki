# Frontend — Index

全部 wiki 页面的目录，按分类组织。每次 ingest 后更新。

## 导航

- [[home]] — 顶层综合、核心洞见与开放问题
- [[log]] — 活动日志

---

## 第一期：WebAssembly（`wiki/wasm/`）

按**建议阅读顺序**排列。

### 基础层

- [[wasm/what-is-wasm]] — Wasm 是什么、四个设计目标、为什么不用 asm.js/LLVM bitcode、版本时间线、三个常见误解
- [[wasm/core-model]] — Module/Instance/Memory/Table 四件套、多重性、结构化栈机、两套栈、验证与 trap、index space
- [[wasm/text-format-wat]] — S-表达式语法、栈机指令、import 两级命名空间、data/elem 段、**table 与 call_indirect 的完整推导**、指令命名规律

### 数据与内存

- [[wasm/types-and-abi]] — 全部值类型与裁剪理由、`externref` 为什么不透明、**i64↔BigInt**、JS↔Wasm 类型映射全表、复合数据的三种传递模式、wasm32/wasm64 数据模型
- [[wasm/memory-model]] — 64KiB 页、**grow 与 detach 陷阱**、import vs export memory、bulk memory 指令、多内存、共享内存、为什么 maximum 可选

### 与 JS 集成

- [[wasm/js-interop]] — JS API 全景、四种加载方式对比、Module 跨 Worker 复用、importObject 结构、动态链接、JSPI、异常处理、ESM 集成现状

### 工具链

- [[wasm/toolchain-rust]] — 三层工具分工、`--target` 选择表、wasm-bindgen 胶水生成原理、async 互操作、体积优化清单
- [[wasm/toolchain-emscripten]] — 四种调用方向（ccall/cwrap/EM_JS/Embind）、数据搬运模式、优化 flag 速查、C++ 异常与 RTTI 的历史包袱、Asyncify、pthreads 约束、**与 Rust 路线的选型对比**
- [[wasm/build-integration]] — Vite/Webpack 集成、服务器 MIME 与缓存、加载策略、编译缓存、CSP、TypeScript、体积治理
- [[wasm/debugging]] — 三个层次的调试体验、DevTools、wabt/Binaryen 命令速查、体积分析、**常见症状的排查路径表**

### 深度与工程

- [[wasm/performance]] — 快在哪的四个原因、固有开销清单、**启动成本拆解**、跨边界成本、SIMD、体积优化、profiling
- [[wasm/threads-and-workers]] — 共享内存 + 原子指令 + Worker、**COOP/COEP 的部署代价**、单构建做不到降级、Emscripten pthreads 全部约束、务实的架构建议
- [[wasm/security-sandbox]] — 两层安全目标、**CFI 免费获得的原理**、Wasm 没消灭的问题、trap 清单、**不确定性七来源**、为什么没有 fast-math
- [[wasm/proposals-and-versions]] — Wasm 没有版本号、六个 Phase、**1.0/2.0/3.0 完整特性表**、GC 为什么重要、活跃 proposal 名单、feature detection 怎么做

### 应用

- [[wasm/use-cases]] — 官方用例清单、**Figma/Photoshop/ffmpeg.wasm/Squoosh/SQLite/Pyodide 等真实案例**、四类有优势的场景、五类不擅长的场景
- [[wasm/beyond-browser]] — Wasm 不定义 API 的设计、WASI 三代演进、Component Model、主流运行时对比、服务端/插件/区块链场景、可移植性硬性假设

---

## 面试准备（`wiki/interview/`）

- [[interview/roadmap]] — 三条路径（2 小时 / 一周 / 深入）、逐日安排与动手任务、**按面试轮次准备什么**、最容易被问倒的五个点
- [[interview/question-bank]] — **21 题分三档**，每题带 30–60 秒口述版参考答案 + 加分点
- [[interview/cheatsheet]] — 关键数字、类型映射、代码模板、五个必背的坑、WAT 语法速记、版本里程碑、工具命令

---

## 分析与决策（`wiki/analysis/`）

- [[analysis/when-to-use-wasm]] — **决策树**、五个必须问的问题、更简单的替代方案对照表、分场景速查表、渐进落地路径、**面试怎么答这道题**

---

## 源材料导读（`wiki/sources/`）

- [[sources/mdn-webassembly-docs]] — MDN WebAssembly 文档（22 文件）：覆盖什么、局部陈旧点、不覆盖什么
- [[sources/wasm-design-repo]] — WebAssembly/design 仓库（21 文件）：**深度题的主要来源**，⚠️ 多数文档停留在 MVP 时代
- [[sources/rust-wasm-toolchain-docs]] — wasm-bindgen / wasm-pack 文档（20 文件）：时效性最好的一份
- [[sources/emscripten-docs]] — Emscripten 文档（5 文件）：多线程实操约束的权威来源
- [[sources/wasi-and-proposals]] — WASI + proposals 仓库：**全库时效性的锚点**

---

## 统计

| 分类 | 页数 |
| --- | --- |
| `wasm/` | 16 |
| `interview/` | 3 |
| `analysis/` | 1 |
| `sources/` | 5 |
| 导航页 | 3 |
| **合计** | **28** |

原始源文档：`raw/` 下 68 个文件，约 676 KB。
