# Frontend Knowledge Base

## Domain

前端工程知识库，面向**「快速掌握某个前端专题 + 应付中高级前端面试」**这一目标。

采用**分期建设**：每一期集中攻一个专题，把它做深做透，再开下一期。分类目录按专题增长，
`interview/` `sources/` `analysis/` 三个横切分类跨期复用。

| 期次 | 主题 | 分类目录 | 状态 |
| --- | --- | --- | --- |
| 第一期 | WebAssembly | `wiki/wasm/` | 已完成首轮建设（2026-08-10） |
| 后续 | 浏览器渲染 / JS 引擎 / 构建工具 / 性能 / 框架原理 | 待定 | 未开始 |

**读者画像**：资深前端工程师，JS/TS、构建工具链、浏览器 API 熟练；对新专题有零散认知，
缺少体系化的原理层理解和「面试能讲清楚」的表达。因此本库的写作重心是
**「机制说清楚 + 一句话能答出来」**，而不是 API 手册。

## Conventions

### 语言

- 正文中文，技术术语保留英文原文（如 linear memory、trap、stack machine、externref、proposal）。
- 不写"教程流水账"。每页回答的是**「它是什么 / 为什么这么设计 / 面试怎么答 / 什么时候不该用」**。

### 面试导向标记

每个概念讲清原理后，用引用块给出面试落点：

```markdown
> **面试落点**：<考官真正想听到的一两句话>
```

复习时只扫这些块即可过一遍全库。这是本库最重要的写作约定。

对容易答错、答浅的地方另用：

```markdown
> **⚠️ 常见误答**：<错误说法> —— 实际是 <正确说法>。
```

### 代码示例规范

- 代码块必须标注语言：`js` / `wat` / `rust` / `c` / `bash` / `html`。
- WAT 示例尽量**完整可 `wat2wasm`**，不写残缺片段；确实是片段时用 `;; ...` 标出省略。
- JS 侧示例用现代写法（`instantiateStreaming`、顶层 `await`、ES module），不用过时的 XHR 写法
  （除非正在讲历史演进）。
- 反面示例显式标注 `// ❌` / `// ✅`。

### 版本与时效

- WebAssembly 相关内容以 **Wasm 3.0**（2025 年定稿，含 GC / 异常处理 / memory64 /
  JS String Builtins）为现状基线。
- 引用 proposal 时必须标注**阶段**（Phase 0–5）与是否已进 spec，不要把 Phase 1 提案写成现状。
- 源文档里的过时表述（如 design 仓库里写"MVP 只支持单返回值""未来会有 SIMD"）要**显式修正**，
  按今天的现状写，并可注明"该表述来自早期设计文档"。

### Page Format

Every wiki page should have YAML frontmatter:

```yaml
---
title: Page Title
date: YYYY-MM-DD
tags: [tag1, tag2]
sources: [mdn-wasm/guides/concepts.md]
---
```

`sources` 填 `raw/` 下的相对路径；纯综合、无直接源文档的页面填 `[]`。

### Wikilinks

Use `[[category/page-name]]` for cross-references (include the subdirectory path under `wiki/`),
例如 `[[wasm/memory-model]]`。

**不使用序号前缀**，文件名为英文 kebab-case slug——序号会随知识重组而失效，而 slug 稳定。
页面的阅读顺序由 `wiki/index.md` 和 `wiki/interview/roadmap` 给出。

### Categories

Each category maps to a subdirectory under `wiki/`:

- **wasm** (`wiki/wasm/`): 第一期专题 —— WebAssembly 的定位与历史、核心执行模型、文本格式、
  类型系统与 ABI、线性内存、JS 互操作、Rust / Emscripten 工具链、构建集成、性能、
  多线程、安全沙箱、proposal 与版本演进、调试、真实落地案例
- **interview** (`wiki/interview/`): 学习路线图、分档高频题库与参考答案、速查卡
- **sources** (`wiki/sources/`): `raw/` 下原始源文档的摘要与导读（讲了什么、覆盖哪些考点、有什么局限）
- **analysis** (`wiki/analysis/`): 对比、决策框架、综合与查询结果

后续期次新增专题目录（如 `wiki/rendering/`、`wiki/engine/`），不要把新专题塞进 `wasm/`。

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志

## Operations

### Ingest（本库补充约定）

抓取官方文档作为源材料时：

1. 用 `curl` 从 `raw.githubusercontent.com` 拉原始 markdown/rst 到 `raw/<project>/` 下，
   保持可辨识的目录结构（MDN 内容在 `mdn/content` 仓库里，路径 `files/en-us/<topic>/`）。
2. 在 `wiki/sources/` 建一页导读，注明**这份源材料的时效性问题**（design 仓库多数文档停留在
   MVP 时代，Emscripten/wasm-bindgen 文档是滚动更新的）。
3. 把知识点**拆进专题分类页**，不要照抄源文档结构。
4. 一个源材料通常横跨多页；一次 ingest 要把受影响的页全部更新，再更 `index.md` 与 `log.md`。

### Query

先读 `wiki/index.md` 定位，再进具体页。若答案对面试准备有普适价值，落成
`wiki/analysis/` 或 `wiki/interview/` 新页，不要只留在对话里。
