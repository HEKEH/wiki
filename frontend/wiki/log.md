# Frontend — Activity Log

Append-only chronological record of wiki activity.

## [2026-08-10] ingest | 建库 + 第一期 WebAssembly 首轮建设

### 建库

- 从 `_template/` 创建 `frontend/`，定位为**分期建设的前端专题面试知识库**
- 写定 `CLAUDE.md`：分类为 `wasm/`（第一期专题）+ `interview/` + `sources/` + `analysis/`
  三个跨期复用的横切分类；约定**不用序号前缀**（改用稳定的 kebab-case slug）；
  约定 **`> **面试落点**`** 和 **`> **⚠️ 常见误答**`** 两种标记块

### 抓取源材料

→ `raw/`，共 68 个文件 / ~676 KB

| 目录 | 来源 | 文件数 |
| --- | --- | --- |
| `raw/mdn-wasm/` | `mdn/content` → `files/en-us/webassembly/` | 22 |
| `raw/wasm-design/` | `WebAssembly/design` + `WebAssembly/proposals` | 21 |
| `raw/wasm-bindgen-doc/` | `rustwasm/wasm-bindgen` + `rustwasm/wasm-pack` | 20 |
| `raw/emscripten-doc/` | `emscripten-core/emscripten` 的 rst 文档 | 5 |
| `raw/wasi-doc/` | `WebAssembly/WASI` | 1 |

### 产出 25 个内容页 + 3 个导航页

- `wasm/` 16 页：what-is-wasm、core-model、text-format-wat、types-and-abi、
  memory-model、js-interop、toolchain-rust、toolchain-emscripten、build-integration、
  performance、threads-and-workers、security-sandbox、proposals-and-versions、
  debugging、use-cases、beyond-browser
- `interview/` 3 页：roadmap（三条路径）、question-bank（21 题分三档）、cheatsheet
- `analysis/` 1 页：when-to-use-wasm（决策树 + 面试答法）
- `sources/` 5 页：五份源材料的导读与**时效性评估**
- 导航：home（六条核心洞见 + 六个开放问题）、index、log

### 关键判断记录

1. **现状基线定为 Wasm 3.0**，依据是 `raw/wasm-design/proposals-finished.md`
   （记录到 2025-07-23 WG 会议：Exception handling / JS String Builtins / Memory64 进 3.0）
2. **显式修正了 design 仓库的过时表述**。该仓库多数文档停留在 MVP 时代（2015–2017），
   "未来会加 SIMD / 线程 / GC / 多返回值"等说法早已实现。
   同样修正了 MDN 里"最多 1 个返回值""每个实例只允许一个 table"和
   Emscripten pthreads 文档里"只能在 Firefox Nightly 跑"的陈旧表述
3. **`raw/wasm-design/` 里 6 个文件已被上游掏空**（Semantics/Modules/BinaryEncoding/
   JS/MVP/TextFormat 只剩一句"见规范文档"），已在 [[sources/wasm-design-repo]] 中标注
4. 案例页的产品信息（Figma/Photoshop/ffmpeg.wasm 等）是业界公开信息，
   无直接源文档，`sources: []`

**未解决**：见 [[home]] 的开放问题一节（缺可运行 demo、缺自测的量化 benchmark、
Wasm GC 实际用法未展开、Component Model 只提了名字）。

## [2026-08-10] lint | 第一期首轮建设后的自查

### 结构性检查（全部通过）

- 28 个 wikilink 全部指向存在的页面，无死链
- 25 个内容页 frontmatter 完整（title / date / tags / sources）
- 所有 `sources:` 路径都能在 `raw/` 下找到对应文件
- 无孤儿页（每页都有入链）
- home / index / log 三处页数统计已对齐（wasm 16 + interview 3 + analysis 1 + sources 5 + 导航 3 = 28）

### 修正的内容问题

| 严重度 | 位置 | 问题 | 处理 |
| --- | --- | --- | --- |
| 🔴 高 | `wasm/proposals-and-versions` | SIMD 特性检测的字节数组是错的：section size 声明 10 实际 9、body size 声明 8 实际 7，且函数签名声明返回 `v128` 但函数体 `drop` 掉了值。**该代码在任何引擎上都返回 `false`**，看起来像"引擎不支持" | 删除手写字节数组，改为讲原理（用 WAT 描述待检测模块）+ 明确警告不要手写 + 指向 `wasm-feature-detect` |
| 🟠 中 | `interview/cheatsheet` | WAT 综合示例里 `(elem (i32.const 0) $f1 $f2)` 引用了未定义的 `$f1`/`$f2`，无法 `wat2wasm`，违反本库自己的约定 | 补上两个函数定义 |
| 🟡 低 | `wasm/security-sandbox` | 内部张力：一处说"没有栈 canary"，另一处又推荐 `-fstack-protector-strong`；且源文档说 SSP "not needed" | 改为"默认不开"，并补一段说明区别：防控制流劫持不需要它（返回地址本来就打不到），防影子栈数据被写坏需要它 |
| 🟡 低 | `wasm/performance` | 分层编译混用 V8 与 Wasmtime 术语（Liftoff/TurboFan 是 V8，Cranelift 是 Wasmtime，且 Wasmtime 基线是 Winch） | 按引擎分开写 |
| 🟡 低 | `wasm/beyond-browser` | 称 Wasmtime 为"参考实现"——规范的参考实现其实是 `WebAssembly/spec` 里的 OCaml 解释器 | 改为"最主流的独立运行时"，并补注说明真正的参考实现是什么 |
| 🟡 低 | `wasm/build-integration` | 断言"小于 `assetsInlineLimit` 的 wasm 可能被内联"——各打包器策略不一致且随版本变，无法离线核实 | 改为"直接看 `dist/` 产物"的可执行建议，不断言具体阈值行为 |
| 🔵 nit | `wasm/js-interop`、`interview/cheatsheet` | "选项菜单"式 JS 块里 `const module` / `const instance` 重复声明，整块粘贴无法运行 | 拆成独立代码块 / 改名 |

### 补充的标注

- `wasm/text-format-wat`：补注 MDN"每个模块只允许一张 table"已过时（reference-types 之后支持多 table，
  `call_indirect` 可显式指定 table 索引）
- `wasm/security-sandbox`：明确标注"Wasm 里没有 ASLR、堆布局确定 → 堆溢出比原生更易稳定利用"
  这一段**不在 `raw/` 源文档里**，是本库推论 + 业界公开研究（USENIX Security '20）
- `wasm/beyond-browser`：明确标注"微秒级 vs 百毫秒级冷启动"是**厂商宣称的量级**，
  不在源文档里且依赖测量口径，建议面试里只说量级不报数字

### 已知未解决

- ⚠️ **WAT 示例未经机器校验**。当前环境没有 node / wabt，19 个 WAT 代码块是逐块人工审查的
  （已修掉 cheatsheet 那个确定的错误）。下次有工具时应跑一遍
  `wat2wasm` 批量验证，多内存那个例子需要 `--enable-multi-memory`。
- 其余开放问题见 [[home]]。

## [2026-08-10] lint | 第二轮自查（换角度：非 JS 代码块 + 无源断言）

上一轮主要看 JS/WAT 和结构，这轮改看 **bash/rust/c/toml 代码块**和**没有源文档支撑的断言**。

### 修正的内容问题

| 严重度 | 位置 | 问题 | 处理 |
| --- | --- | --- | --- |
| 🟠 中 | `wasm/toolchain-rust` | 推荐 `wee_alloc` —— 该 crate 已被 rustwasm **归档、不再维护**，且有未修复的内存泄漏问题。2026 年这是过时且有害的建议 | 改为明确劝阻，给出 `lol_alloc` / `talc` 或直接用默认分配器 |
| 🟠 中 | `wasm/toolchain-emscripten` | 两个 C 示例缺必要 include：`int_sqrt` 用了 `sqrt()` 却没 `<math.h>`；`create_buffer` 用了 `malloc`/`uint8_t` 却没 `<stdlib.h>`/`<stdint.h>`（MDN 原文是有的，转写时丢了） | 补齐 include |
| 🟡 低 | `wasm/what-is-wasm` | 版本时间线里 2019 标了"W3C Recommendation"，2.0/3.0 只标年份，读起来像三者同级 | 明确只有 1.0 是 W3C Recommendation；2.0/3.0 是规范文档版本号，年份是最后一批 proposal 过 WG 的时间，不是发布日 |
| 🟡 低 | `wasm/toolchain-rust` | `spawn_local` / `future_to_promise` 只写了名字。源文档用的是 `js_sys::futures::` 路径，但那只在较新版本存在，读者照抄会编不过 | 补注：稳妥写法是 `wasm_bindgen_futures::spawn_local` |
| 🟡 低 | `wasm/use-cases` | "Photoshop 几百万行 C++"是我编的量级，Adobe 未公开 | 改为不带数字的表述，并注明别在面试里编数字 |
| 🟡 低 | `wasm/use-cases` | Google Earth 只写了"摆脱 NPAPI"，跳过了 NaCl 这一步；且把 UseCases 里的条目说成了"设计目标" | 补全 NPAPI → NaCl → Wasm 路径，并改为准确引用"官方用例清单" |
| 🟡 低 | `wasm/use-cases` | "Pyodide 首次加载动辄十几 MB"——取决于加载哪些包，数字不稳 | 改为"数 MB 运行时 + 按需加载包" |
| 🔵 nit | `wasm/what-is-wasm` | 常见误解清单里三个连续 blockquote 之间只隔空行（MD028），部分渲染器会合并成一坨 | 每条加 `###` 小标题分隔 |

### 口径统一

"Wasm 冷启动微秒级 vs 容器百毫秒级"原本在 4 个页面各说一遍。现统一为
**"快好几个数量级"**，具体数字只在 `wasm/beyond-browser` 保留并标注为厂商口径。
`interview/question-bank` 补了一句提醒：面试里说数量级就够了，报具体数字容易被追问"你测过吗"。

### 结构性检查（全部通过）

- 28 个 wikilink 无死链；25 页 frontmatter 完整；无孤儿页
- 无标题跳级；无相邻 blockquote
- 所有表格列数一致（`types-and-abi` 那处是检查脚本对 GFM 转义 `\|` 的误报，实际正确）

### 仍然未解决

- ⚠️ **WAT 示例依旧未经机器校验**（环境无 node / wabt）。这是目前最大的残留风险。
- 其余开放问题见 [[home]]。

## [2026-08-10] query | GC 是什么？为什么设计成没有字符串、没有对象？

问题来自 `wasm/what-is-wasm` 那句"不偏向任何语言族：所以 core wasm 里没有字符串、
没有对象、没有 GC"——这句话压缩了两个独立的设计决策，展开后是一道高区分度的面试题。

**发现的缺口**：`wasm/types-and-abi` 只写了结论（"复杂类型由源语言编译器构造"），
没写理由；`wasm/proposals-and-versions` 讲了 GC 的意义，但没澄清最容易被误解的一点。

**落库**：

- `wasm/types-and-abi` 新增「为什么没有字符串、没有对象？」一节 ——
  核心是**这两样东西不存在中立定义**（六种语言的字符串表示对照表 + 对象模型互不兼容），
  内建任何一种都违背"不偏向任何语言族"；另附三条辅证（虚拟 ISA 定位 /
  有对象就必须有 GC 而 MVP 排除了 GC / 小类型系统利于单遍验证）。
  面试落点写的是**"没有对象"和"3.0 加了 GC"是同一问题的两端** ——
  GC proposal 交出去的是"内存管理"层，不是"对象模型"层；
  JS String Builtins 同理，规范层面 Wasm 至今没有字符串类型。
  引用了 FAQ 的 *"a walled-off world of their own"*。
- `wasm/proposals-and-versions` 新增「GC 不回收线性内存」小节 + 对照表。
  这是最常见的误解：**线性内存永远不会被 GC**，引擎看到的只有字节，
  不知道哪些构成对象。GC 加的是另一套引擎管理的堆类型（`struct`/`array`），
  这也正是"Rust/C++ 不受影响"的真正原因。

无新增页面，`index.md` 不变。链接与 MD028 检查通过。
