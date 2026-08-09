---
title: "WSGI 与 ASGI：Python Web 的底层协议"
date: 2026-08-07
tags: [WSGI, ASGI, uvicorn, gunicorn, starlette, 协议]
sources: ["fastapi-doc/deployment/concepts.md", "interview-python-cn.md"]
---

# WSGI 与 ASGI：Python Web 的底层协议

**面试必问的架构题**：「FastAPI 和 Flask/Django 有什么本质区别？」
「uvicorn 和 gunicorn 是什么关系？」——答案都在这一层。

## 1. 为什么需要这层协议

Python Web 生态很早就把「**HTTP 服务器**」和「**Web 框架**」解耦了：

```text
浏览器 ──HTTP──> Web 服务器（uvicorn/gunicorn/nginx）
                     │
                     │  WSGI / ASGI 协议（一个函数签名的约定）
                     ▼
                 Web 框架（Flask/Django/FastAPI）
                     │
                     ▼
                 你的业务代码
```

好处：任何框架能跑在任何兼容服务器上。这是 PEP 333（WSGI，2003）的贡献。

> 前端类比：≈ Node 里 `http.createServer((req, res) => {...})` 的回调签名约定，
> Express/Koa/Fastify 都基于它。只是 Python 把这个约定写成了 PEP 标准。

## 2. WSGI（同步，PEP 3333）

**一个 WSGI 应用就是一个可调用对象**：

```python
def app(environ, start_response):
    """
    environ: dict，包含请求的所有信息（CGI 风格的键）
    start_response: 回调，用来发送状态码和响应头
    返回: 一个可迭代的 bytes（响应体）
    """
    status = "200 OK"
    headers = [("Content-Type", "text/plain; charset=utf-8")]
    start_response(status, headers)
    return [b"Hello, WSGI"]

# 用标准库直接跑起来
from wsgiref.simple_server import make_server
make_server("", 8000, app).serve_forever()
```

**根本限制**：这个签名是**同步阻塞**的。

- 一个请求 = 一个线程（或进程）**从头占到尾**。
- 并发数 ≈ worker 线程数。1000 个并发连接需要 1000 个线程 → 内存和切换成本爆炸。
- **无法支持 WebSocket、SSE、HTTP/2 push**——协议里根本没有"长连接双向通信"的位置。

WSGI 服务器：**gunicorn**（最流行）、uWSGI、waitress（纯 Python，Windows 友好）、mod_wsgi。
WSGI 框架：Flask、Django（4.0 前）、Bottle、Pyramid。

## 3. ASGI（异步，Asynchronous Server Gateway Interface）

```python
async def app(scope, receive, send):
    """
    scope:   dict，连接信息（type: 'http' / 'websocket' / 'lifespan'）
    receive: async 可调用，接收事件（请求体分块、ws 消息、断开）
    send:    async 可调用，发送事件（响应头、响应体分块）
    """
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello, ASGI",
    })
```

三处关键改进：

| | WSGI | ASGI |
|---|---|---|
| 函数 | 同步 | **`async def`** |
| 返回 | 一次性返回响应体 | **事件流**（`receive`/`send` 可多次调用） |
| 协议 | 只有 HTTP | **HTTP + WebSocket + lifespan**（`scope["type"]`） |
| 并发模型 | 线程/进程 | **协程**（单线程数万连接） |
| 流式 | 有限 | 原生支持（分块 send） |

**lifespan** 是 ASGI 独有的：服务器启动/关闭时会给应用发 `lifespan.startup` /
`lifespan.shutdown` 事件——这就是 FastAPI `lifespan` 的底层。

ASGI 服务器：**uvicorn**（基于 uvloop + httptools，最流行）、hypercorn（支持 HTTP/2、HTTP/3）、
daphne（Django Channels 出品）、granian（Rust 写的，新秀）。
ASGI 框架：**Starlette**、**FastAPI**（构建在 Starlette 之上）、Django 3.0+（异步视图）、
Litestar、Quart（Flask 的异步版）、Sanic。

## 4. FastAPI 的技术栈定位

```text
┌──────────────────────────────────────┐
│  你的代码：路由函数 + Pydantic 模型     │
├──────────────────────────────────────┤
│  FastAPI    —— 依赖注入、参数解析、OpenAPI 生成、校验集成
├──────────────────────────────────────┤
│  Starlette  —— 路由、请求/响应对象、中间件、WebSocket、后台任务、TestClient
├──────────────────────────────────────┤
│  Pydantic   —— 数据校验与序列化（v2 核心是 Rust）
├──────────────────────────────────────┤
│  ASGI 协议
├──────────────────────────────────────┤
│  uvicorn    —— 事件循环（uvloop）+ HTTP 解析（httptools）
├──────────────────────────────────────┤
│  asyncio / uvloop（libuv）
└──────────────────────────────────────┘
```

> **面试落点**：「FastAPI 为什么快？」标准答案三条：
> ① 基于 **ASGI + asyncio**，I/O 密集场景下单进程能处理海量并发连接，不再是"一请求一线程"；
> ② 服务器层用 **uvicorn（uvloop + httptools，都是 C/Cython）**；
> ③ 校验层用 **Pydantic v2（核心是 Rust 编写的 pydantic-core）**。
> 补一句「FastAPI 本身只是胶水层，快的是它站的这几个肩膀」显得很清醒。

## 5. gunicorn vs uvicorn（高频混淆点）

| | gunicorn | uvicorn |
|---|---|---|
| 协议 | WSGI（也可作为进程管理器） | **ASGI** |
| 角色 | **进程管理器 + WSGI 服务器** | ASGI 服务器 |
| 强项 | 成熟的多进程管理、优雅重启、worker 监控 | 高性能异步 I/O |

**生产上的经典组合**（用 gunicorn 管进程，uvicorn 处理协议）：

```bash
gunicorn app.main:app \
  -w 4 \
  -k uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 60 \
  --max-requests 1000 --max-requests-jitter 100 \
  --access-logfile - --error-logfile -
```

**但从 uvicorn 0.30 / FastAPI 官方文档的现代建议看**，容器化部署更推荐：

```bash
# 容器里跑单进程，进程复制交给 Kubernetes / Docker Swarm 的副本数
uvicorn app.main:app --host 0.0.0.0 --port 8000

# 或用 fastapi CLI（0.111+）
fastapi run app/main.py --port 8000
```

理由：**容器编排系统已经在做进程/副本管理了**，再套一层 gunicorn 是重复且会让
健康检查、优雅关闭、资源限制变复杂。见 [[web/fastapi-production]]。

## 6. WSGI ↔ ASGI 互操作

```python
# 在 ASGI 应用里挂载一个 WSGI 应用（渐进式迁移 Django/Flask 的标准手段）
from fastapi import FastAPI
from fastapi.middleware.wsgi import WSGIMiddleware
from flask import Flask

flask_app = Flask(__name__)
app = FastAPI()
app.mount("/legacy", WSGIMiddleware(flask_app))
```

反过来（在 WSGI 里跑 ASGI）需要 `asgiref` 的 `AsgiToWsgi`，但会丢失异步优势。

Django 的异步之路也走的是 ASGI：`asgiref.sync.sync_to_async` / `async_to_sync`
用来在两个世界之间桥接。

## 7. ASGI 中间件

ASGI 中间件就是**包裹另一个 ASGI 应用的 ASGI 应用**（洋葱模型，和 Koa 一样）：

```python
class TimingMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        start = time.perf_counter()

        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                elapsed = time.perf_counter() - start
                message["headers"].append(
                    (b"x-process-time", f"{elapsed:.4f}".encode())
                )
            await send(message)

        await self.app(scope, receive, send_wrapper)

app.add_middleware(TimingMiddleware)
```

Starlette 还提供了更简单的 `BaseHTTPMiddleware`（用 `@app.middleware("http")` 装饰），
但它有性能开销和流式响应的坑——**高性能场景写纯 ASGI 中间件**。

## 8. CGI / FastCGI（历史，偶尔被问）

| 时代 | 方案 | 问题 |
|---|---|---|
| 1993 | **CGI** | 每个请求 fork 一个进程，极慢 |
| 1996 | **FastCGI** | 常驻进程池，快很多，但接口复杂 |
| 2003 | **WSGI**（PEP 333） | Python 专用、简单、生态统一 ✅ |
| 2018 | **ASGI** | 支持异步与 WebSocket ✅ |

> **面试落点**：中文题库里的"CGI 和 WSGI 的区别"——
> **CGI 是语言无关的进程级协议（每请求一进程）；WSGI 是 Python 专用的函数级接口
> （常驻进程，一次调用一个函数），性能高出几个数量级。**

## 相关

- [[web/fastapi-core]] —— FastAPI 的核心用法
- [[web/fastapi-production]] —— worker 模型与部署
- [[concurrency/asyncio-fundamentals]] —— ASGI 依赖的事件循环
- [[concurrency/concurrency-models]] —— 一请求一线程 vs 协程
- [[interview/question-bank-web]] —— Web 架构面试题
