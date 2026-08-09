---
title: "源：FastAPI 与 Pydantic 官方文档"
date: 2026-08-07
tags: [源导读, FastAPI, Pydantic, 官方文档]
sources: ["fastapi-doc/async.md", "fastapi-doc/tutorial/first-steps.md", "pydantic-doc/models.md"]
---

# 源：FastAPI 与 Pydantic 官方文档

**抓取日期**：2026-08-07，来自 `fastapi/fastapi` 与 `pydantic/pydantic` 仓库的 `docs/`。

## 抓取清单

```text
raw/fastapi-doc/
├── async.md                       ★ 并发与 async/await 的长篇解释（24 KB）
├── python-types.md                类型注解入门
├── virtual-environments.md        虚拟环境（现在推荐 uv）
├── tutorial/
│   ├── first-steps.md  path-params.md  query-params.md  body.md
│   ├── response-model.md          ★ 入参/出参模型分离
│   ├── dependencies/index.md      ★ 依赖注入
│   ├── dependencies/dependencies-with-yield.md  ★ yield 依赖的执行顺序
│   ├── security/oauth2-jwt.md     ★ JWT 鉴权完整实现
│   ├── sql-databases.md           SQLModel 版数据库教程
│   ├── testing.md  bigger-applications.md  background-tasks.md
│   ├── handling-errors.md  middleware.md
├── advanced/
│   ├── settings.md                ★ pydantic-settings + lru_cache
│   ├── events.md                  ★ lifespan（取代 on_event）
│   ├── async-tests.md  websockets.md
└── deployment/
    ├── concepts.md                ★ 部署概念（HTTPS/进程/重启/复制/内存）
    └── docker.md                  ★ Dockerfile 最佳实践

raw/pydantic-doc/
├── models.md         (59 KB) 模型定义、嵌套、泛型、序列化
├── validators.md     (30 KB) field_validator / model_validator / before-after
├── fields.md         (28 KB) Field 的全部参数、alias、约束
└── performance.md    性能建议
```

## FastAPI 文档的特别之处

**它是我见过质量最高的框架文档之一**，值得完整读一遍（不是查，是读）。特点：

1. **教程式而非参考式**——每个概念都有可运行的完整示例和渐进式的"为什么"。
2. **`async.md` 是一篇独立的并发科普**——用"汉堡店排队"的比喻讲清了
   并发 vs 并行、什么时候用 `async def`、什么时候用 `def`。
   **这一篇直接决定了你会不会写出阻塞事件循环的代码。**
3. **对新手的常见误区有专门澄清**（比如"用了 async 不一定更快"）。
4. 多语言版本（含中文），但**英文版更新最快**。

## 关键结论的出处

| 本库中的结论 | 源文档 |
|---|---|
| `def` 路由被自动放进线程池，`async def` 在事件循环跑 | `async.md` |
| 依赖在同一请求内只执行一次（有缓存） | `tutorial/dependencies/index.md` |
| `yield` 依赖的收尾在**响应发送之后**执行 | `dependencies-with-yield.md` |
| **BackgroundTasks 在 yield 依赖收尾之后跑**（所以不能用请求 session） | `dependencies-with-yield.md` |
| `lifespan` 取代已弃用的 `@app.on_event` | `advanced/events.md` |
| Settings + `@lru_cache` 的配置注入模式 | `advanced/settings.md` |
| 容器化部署推荐单进程 + 编排层做复制 | `deployment/concepts.md`、`docker.md` |
| 依赖分层 COPY 以利用 Docker 缓存 | `deployment/docker.md` |
| 官方推荐用 uv 管理项目 | `virtual-environments.md` |

## Pydantic 文档

**v2 的文档明确标注了从 v1 迁移的所有变更**，这是本库
[[web/pydantic]] 里那张 v1→v2 对照表的来源。

重点章节：

- `models.md`：模型定义、嵌套模型、泛型模型、`model_config` 全部选项、
  `from_attributes`（原 orm_mode）、序列化控制
- `validators.md`：`field_validator` / `model_validator` 的 `mode="before"/"after"` 语义、
  `ValidationInfo`、序列化器
- `fields.md`：`Field()` 的全部参数、alias 与 `alias_generator`（snake_case ↔ camelCase）
- `performance.md`：**官方性能建议**（用 `model_validate_json` 而非 `json.loads` + validate、
  避免在函数内重复定义模型、`TypeAdapter` 复用）

## 在本库里的消化

| 本库页面 | 主要来源 |
|---|---|
| [[web/fastapi-core]] | first-steps / path-params / query-params / body / response-model |
| [[web/fastapi-di]] | dependencies/index + dependencies-with-yield |
| [[web/pydantic]] | pydantic 全部四篇 |
| [[web/fastapi-async-db]] | sql-databases（本库改用 SQLAlchemy 2.0 async 而非 SQLModel） |
| [[web/fastapi-auth]] | security/oauth2-jwt |
| [[web/fastapi-architecture]] | bigger-applications / events / middleware / handling-errors / background-tasks |
| [[web/fastapi-testing]] | testing / async-tests |
| [[web/fastapi-production]] | deployment/concepts + deployment/docker |
| [[concurrency/concurrency-models]] | async.md |

## 本库与源文档的差异（有意为之）

| 主题 | 官方教程 | 本库 |
|---|---|---|
| 数据库 | 用 **SQLModel**（作者自己的库，同步为主） | 用 **SQLAlchemy 2.0 async**——生产项目的主流，且能讲清 session/N+1/事务 |
| 参数声明 | 教程里两种写法混用 | 统一用 **`Annotated`**（官方推荐的现代写法） |
| 项目结构 | `bigger-applications` 只给了基本示例 | 补充了完整的**分层架构 + 垂直切分**方案 |
| 部署 | 概念性讲解 | 补充了 K8s 优雅关闭、preStop、连接池容量计算等实操细节 |
| 面试视角 | 无 | 每页都标注了**面试落点** |

## 相关

- [[web/fastapi-core]] 等 `web/` 全部页面 —— 主要消化页
- [[web/wsgi-asgi]] —— 补充了官方文档没细讲的协议层
- [[interview/question-bank-web]] —— Web 题库
