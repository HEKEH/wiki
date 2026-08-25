# Python Knowledge Base

## Domain

Python 语言与后端工程知识库，面向**「前端工程师转 Python + 冲刺高难度 Python 面试」**这一具体目标。

覆盖：语言核心机制（数据模型、描述符、元类、生成器、装饰器）、CPython 内部原理（引用计数、GC、
GIL、字节码、内存模型）、并发与 asyncio、标准库与数据结构、**FastAPI + Pydantic + SQLAlchemy
后端技术栈**、工程实践（打包、测试、类型检查、性能），以及针对前端背景的 JS/TS ↔ Python 对照。

**读者画像**：熟悉 JavaScript/TypeScript、Node.js、异步编程、npm 生态；对 Python 语法有基本了解，
但缺少 CPython 底层、GIL/GC、Python 工程惯例、Python 后端栈的系统知识。因此本库大量使用
**「JS 里的 X ≈ Python 里的 Y，但差别在 Z」** 的对照式讲解——这是本库最重要的写作约定。

## Conventions

### 语言

- 正文中文，技术术语保留英文原文（如 descriptor、coroutine、GIL、dunder）。
- 每个概念页必须有**可运行的代码示例**，而不只是文字描述。这是本库的硬性要求。

### 代码示例规范

- 代码块标注语言（```python / ```bash / ```console）。
- 示例尽量**自包含可运行**；有输出的用 `#=>` 注释标出预期结果：

  ```python
  a = [1, 2, 3]
  b = a[:]
  a is b          #=> False
  a == b          #=> True
  ```

- 交互式会话用 `>>>` 前缀，脚本片段不用。
- 面向 Python **3.12+** 编写（必要时标注版本差异，如 3.13 free-threading、3.11 TaskGroup）。
- 反面示例必须显式标注 `# ❌ 错误写法` / `# ✅ 正确写法`。

### 面试导向标记

概念页在讲清原理之后，用引用块给出面试落点：

```markdown
> **面试落点**：<考官真正想听到的一两句话>
```

这样可以只扫这些块做快速复习。

### Page Format

Every wiki page should have YAML frontmatter:

```yaml
---
title: Page Title
date: YYYY-MM-DD
tags: [tag1, tag2]
sources: [source-file-1.md]
---
```

`sources` 填 `raw/` 下的相对路径；纯综合、无直接源文档的页面填 `[]`。

### Wikilinks

Use `[[category/page-name]]` for cross-references (include the subdirectory path under `wiki/`).
不使用序号前缀，文件名为英文 kebab-case slug。

### Categories

Each category maps to a subdirectory under `wiki/`:

- **language** (`wiki/language/`): 语言核心机制 —— 数据模型、对象与可变性、作用域闭包、装饰器、
  迭代器生成器、上下文管理器、类与 MRO、描述符、元类、类型注解、异常、数据类、推导式与函数式
- **internals** (`wiki/internals/`): CPython 实现原理 —— 对象模型、垃圾回收、GIL、字节码执行、内存模型
- **stdlib** (`wiki/stdlib/`): 内置数据结构的底层实现与复杂度、常用标准库工具箱
- **concurrency** (`wiki/concurrency/`): 并发模型选型、threading、multiprocessing、asyncio 原理与实战
- **web** (`wiki/web/`): 后端 Web 栈 —— WSGI/ASGI、FastAPI 核心/依赖注入/鉴权/测试/部署、Pydantic、SQLAlchemy
- **engineering** (`wiki/engineering/`): 工程实践 —— 包管理与虚拟环境、pytest、静态检查工具链、性能剖析
- **bridge** (`wiki/bridge/`): 前端视角的对照页 —— JS↔Python 语法心智、两种事件循环对比、npm↔pip 生态
- **interview** (`wiki/interview/`): 学习路线图、分主题高频题库与参考答案、手撕代码惯用法、经典陷阱题
- **review** (`wiki/review/`): 复习题组 —— 按 roadmap 进度产出的自测卷与批改记录，文件名
  `review-set-NN.md`（两位数字，递增，不复用编号）。每组是**多考点串联的综合题**，
  与 `interview/traps` 的单点速查、`interview/question-bank-*` 的分主题题库分工不同。
  单题结构固定为「题目 / 参考答案 / 解析 / 面试落点」四段，所有输出必须实测并标注 CPython 版本；
  页首用一张表列出「# / 主题 / 涉及页面 / 高频错点」，并回链对应的 [[interview/roadmap]] 天数。

  **题型限定（2026-08-25 起）**：「题目」段只出**可机械判分**的三类题——

  1. **写输出**：给一段代码，预测 stdout 逐字符输出（会抛异常的行写出异常类型与消息）。
  2. **判断题**：给一句论断，判对错。
  3. **选择题**：单选或多选。

  **不再出开放式简答 / 追问题**——例如「为什么这么设计」「说说三者的关系」「给出两种修法」
  「这在工程上有什么真实事故」。理由：作答耗时太长，性价比低于同样时间多做几道输出题。

  注意这**只限制「题目」段，不削减内容**：原本放在追问里的机制解释、设计动机、修法、
  事故案例，一律移进「解析」段由我直接讲清楚，读者读而不答。
  需要考查机制理解时，把它**改写成输出题或判断题**（例如不问「`from None` 时三个字段各是什么」，
  而是给一段打印 `__cause__` / `__context__` / `__suppress_context__` 的代码让人写输出）。
- **sources** (`wiki/sources/`): `raw/` 下原始源文档的摘要与导读
- **analysis** (`wiki/analysis/`): 对比、综合与查询结果

### Special Pages

- `wiki/home.md` — 顶层综合与导航，核心洞见和开放问题
- `wiki/index.md` — 内容目录
- `wiki/log.md` — 活动日志

## Operations

### Ingest（本库补充约定）

抓取 GitHub / 官方文档源码作为源材料时：

1. 用 `curl` 拉 `raw.githubusercontent.com` 上的原始 markdown/rst 到 `raw/<project>-doc/` 下，保持原目录结构。
2. 在 `wiki/sources/` 建一页导读（这份源材料讲了什么、覆盖哪些考点、有什么局限/过时之处）。
3. 把知识点拆进对应的 `language/` `internals/` `web/` 等分类页，而不是照抄源文档。
4. 老源材料（如 2015 年前后的中文面试题库）里的 Python 2 内容要**显式标注为已过时**，不要当现状写。

### Query

回答问题时优先读 `wiki/index.md` 定位，再进具体页。若答案对面试准备有普适价值，落成
`wiki/analysis/` 或 `wiki/interview/` 新页。
