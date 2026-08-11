---
title: 真实落地案例
date: 2026-08-10
tags: [wasm, 案例, 应用]
sources: [wasm-design/UseCases.md, mdn-wasm/guides/existing_c_to_wasm.md, wasm-design/FAQ.md]
---

# 真实落地案例

面试里被问"你了解 Wasm 在哪些地方用了吗"，能举出**具体案例 + 它为什么非 Wasm 不可**
比背官方 use case 列表有用得多。

## 官方设想的用例（design 文档）

浏览器内：图像/视频编辑、游戏（轻量快启动的、AAA 大资源的、混合来源的游戏门户）、
P2P 应用、音乐应用、图像识别、实时视频增强、VR/AR（超低延迟）、CAD、
科学可视化与仿真、交互式教育软件、平台模拟器（DOSBox/QEMU/MAME）、
语言解释器与虚拟机、POSIX 用户态环境、开发者工具、远程桌面、VPN、加密、本地 Web 服务器。

浏览器外：游戏分发、**服务端执行不可信代码**、服务端应用、移动端混合应用、多节点对称计算。

## 有代表性的真实案例

### Figma —— Wasm 让"设计工具上 Web"成立

C++ 写的渲染/几何引擎编译到 Wasm，UI 用 React。
Figma 是最早公开分享 Wasm 收益的产品之一：从 asm.js 迁到 Wasm 后**加载时间大幅缩短**。

**为什么非 Wasm 不可**：设计工具的核心是几何运算和渲染，
JS 的 GC 停顿和不可预测性会直接表现为拖动卡顿。

### Photoshop Web / Adobe

Emscripten 把庞大的 C++ 代码库搬上 Web，配合 WebGL/WebGPU 渲染。
这是"移植既有代码库"的典型：**积累了几十年的 C++ 代码库，用 JS 重写既不现实也不划算**。
（具体代码规模 Adobe 未公开，别在面试里编数字。）

### Google Earth

C++ 三维渲染引擎编译到 Wasm。它的历史路径正好是 Web 原生化的缩影：
**NPAPI 插件 → Native Client（NaCl）→ WebAssembly**。

官方 `UseCases.md` 的用例清单里明确列着一条
"Common NPAPI users, within the web's security model and APIs"——
即**在 Web 的安全模型内，接管原本要靠 NPAPI 插件才能做的事**。

### ffmpeg.wasm

FFmpeg 编译到 Wasm，在浏览器里做音视频转码。

**为什么价值大**：视频**不用上传到服务器**——隐私、带宽、成本三重收益。
代价是产物很大（几十 MB），必须按需加载。

### Squoosh（Google）

图像编解码器（MozJPEG、WebP、AVIF、OxiPNG）编译到 Wasm，浏览器内压缩图片。
MDN 的 libwebp 教程本质就是这个案例的简化版。

**技术要点**：canvas `getImageData()` 拿 RGBA → 拷进线性内存 → Wasm 编码 →
把结果视图拷成独立 `Uint8Array` → `new Blob([result], { type: "image/webp" })`。

### SQLite WASM

官方支持的 Wasm 构建，配合 OPFS（Origin Private File System）在浏览器里跑完整 SQL 数据库。
这是"把成熟 C 库整个搬上 Web"的模板案例。

### Pyodide / Ruby.wasm

把整个 CPython / CRuby 虚拟机编译到 Wasm，在浏览器里跑 Python/Ruby（含 numpy、pandas）。

官方 FAQ 早就点出了这种"编译整个 VM"策略的代价：
**产物体积大、丢失浏览器 devtools 集成、跨语言循环引用无法回收、
错过需要引擎深度集成的优化**。Pyodide 首次加载要拉数 MB 的运行时，
再按需加载各个包（numpy/pandas 这类还会显著加码）——就是这个原因。

### AutoCAD Web / Unity / Unreal

大型 C++ 应用整体编译到 Wasm + WebGL。属于"整个应用就是 Wasm"这一类形态。

### eBay 的条码扫描器

一个常被引用的对照实验：JS 版本的条码识别成功率不够，用 C++ 编译到 Wasm 后
识别率和速度都显著提升。这是"局部替换热点算法"的典型。

### 服务端 / 边缘

- **Fastly Compute@Edge**、**Cloudflare Workers**：用 Wasm 做多租户隔离，
  冷启动比容器快几个数量级
- **Envoy / Istio 的 WasmPlugin**：用 Wasm 写代理插件，不用重编译代理本身
- **Shopify Functions**：让商家用任意语言写业务逻辑，平台用 Wasm 沙箱执行
- **Docker + Wasm**：把 Wasm 作为容器的轻量替代运行时

详见 [[wasm/beyond-browser]]。

## 归纳：Wasm 真正有优势的四类场景

### 1. 移植既有原生代码库

**最稳的收益**。FFmpeg、SQLite、libwebp、OpenCV、PDFium、libarchive……
这些库有几十年的打磨，用 JS 重写既不现实也不明智。

### 2. 计算密集且能大块传数据

图像/视频编解码、压缩、加解密、物理仿真、几何运算、机器学习推理。
共同点：**一次调用做很多工作**，跨边界开销可以忽略。

### 3. 需要性能可预测

游戏、音频处理、实时协作。JS 的 GC 停顿和 deopt 会表现为掉帧，Wasm 没有这个问题。

### 4. 需要沙箱隔离不可信代码

插件系统、多租户平台、边缘计算。这一类**和"快"无关**，
纯粹是要 Wasm 的**能力安全模型**——见 [[wasm/security-sandbox]]。

> **面试落点**：第 4 类最容易被忽略但最能体现深度。
> **Wasm 在服务端的主要卖点不是性能，是隔离性和启动速度**——
> 一个 Wasm 实例的冷启动比容器快好几个数量级（具体数字见 [[wasm/beyond-browser]] 的口径说明）。

## Wasm 不擅长的场景

| 场景 | 为什么不适合 |
| --- | --- |
| DOM 密集操作 | 每次都要绕道 JS，比直接写 JS 慢 |
| 字符串/对象密集的业务逻辑 | 需要不停编解码，得不偿失 |
| 小函数高频调用 | 跨边界成本吃掉全部收益 |
| 简单的数据转换/校验 | JS 引擎优化得已经很好，不值得增加构建复杂度 |
| 首屏关键路径上的小逻辑 | 加载 + 编译的固定成本可能超过节省的执行时间 |

决策框架见 [[analysis/when-to-use-wasm]]。

## 相关页面

- 何时该用：[[analysis/when-to-use-wasm]]
- 性能特征：[[wasm/performance]]
- 非 Web 场景：[[wasm/beyond-browser]]
- C 库移植方法：[[wasm/toolchain-emscripten]]
