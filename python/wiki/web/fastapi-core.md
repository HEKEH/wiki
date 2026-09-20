---
title: "FastAPI 核心：路由、参数与响应"
date: 2026-08-07
tags: [FastAPI, 路由, 参数校验, 响应模型, OpenAPI, async]
sources: ["fastapi-doc/tutorial/first-steps.md", "fastapi-doc/tutorial/path-params.md", "fastapi-doc/tutorial/query-params.md", "fastapi-doc/tutorial/body.md", "fastapi-doc/tutorial/response-model.md"]
---

# FastAPI 核心：路由、参数与响应

FastAPI 的设计哲学一句话：**类型注解即契约**——一个函数签名同时决定了
参数解析、数据校验、序列化、OpenAPI 文档和编辑器补全。

> 前端类比：像是 `zod` schema + Express 路由 + Swagger 自动生成，三合一，
> 而且**只写一次**。这是它相对 Express/Koa 最大的体验优势。

## 1. 最小应用

```python
from fastapi import FastAPI

app = FastAPI(title="My API", version="1.0.0")

@app.get("/")
async def root():
    return {"msg": "hello"}
```

```bash
uvicorn main:app --reload          # 或 fastapi dev main.py（0.111+）
# http://127.0.0.1:8000/docs       Swagger UI（自动生成）
# http://127.0.0.1:8000/redoc      ReDoc
# http://127.0.0.1:8000/openapi.json
```

## 2. 参数的四个来源

FastAPI **根据参数的声明位置和类型自动判断它来自哪里**：

```python
from fastapi import FastAPI, Path, Query, Body, Header, Cookie, Depends
from pydantic import BaseModel

class ItemIn(BaseModel):
    name: str
    price: float

@app.put("/items/{item_id}")
async def update(
    item_id: int = Path(ge=1),                  # ① 路径参数（在 URL 模板里出现）
    q: str | None = Query(None, max_length=50), # ② 查询参数（其余标量类型）
    item: ItemIn = Body(),                      # ③ 请求体（Pydantic 模型）
    x_token: str = Header(),                    # ④ 请求头（下划线自动转连字符）
    session: str | None = Cookie(None),         #    Cookie
):
    ...
```

**判定规则**（记住这条就够了）：

| 参数类型 | 默认来源 |
|---|---|
| 名字出现在路径模板中 | **路径参数** |
| 标量类型（`int`/`str`/`float`/`bool`/`list[str]`） | **查询参数** |
| Pydantic 模型 | **请求体** |
| 显式 `Header()` / `Cookie()` / `Form()` / `File()` | 对应位置 |
| `Depends(...)` | 依赖注入，见 [[web/fastapi-di]] |

### 现代写法：`Annotated`（官方推荐）

```python
from typing import Annotated

@app.get("/items/")
async def read(
    q: Annotated[str | None, Query(max_length=50)] = None,
    user: Annotated[User, Depends(get_current_user)],
):
    ...
```

`Annotated` 把"类型"和"元数据"分开，好处是**默认值回归 Python 语义**、
同一个依赖能被复用成类型别名：

```python
CurrentUser = Annotated[User, Depends(get_current_user)]
DBSession   = Annotated[AsyncSession, Depends(get_session)]

@app.get("/me")
async def me(user: CurrentUser, db: DBSession):    # 极其清爽
    ...
```

> **面试落点**：能说出「`Annotated` 是 FastAPI 0.95+ 的推荐写法，
> 它避免了 `q: str = Query(None)` 这种把默认值写进 `Query()` 的旧模式，
> 让依赖可以被抽成可复用的类型别名」——说明你在用现代写法而不是老教程。

### 校验约束

```python
Path(ge=1, le=1000)
Query(min_length=3, max_length=50, pattern="^fixed")
Query(default_factory=list)
Body(embed=True)                                  # 让单个模型也包一层键
Query(deprecated=True, description="…", example="x")
```

## 3. 响应模型

```python
class UserIn(BaseModel):       # 入参：含密码
    username: str
    password: str
    email: str

class UserOut(BaseModel):      # 出参：不含密码 ★
    username: str
    email: str

@app.post("/users/", response_model=UserOut, status_code=201)
async def create(user: UserIn):
    saved = await save(user)
    return saved               # 即使返回了完整对象，也只会序列化出 UserOut 的字段
```

**`response_model` 做三件事**：过滤字段（**安全边界**）、校验输出、生成 OpenAPI schema。

```python
# 也可以用返回类型注解（更 Pythonic，0.89+）
@app.post("/users/")
async def create(user: UserIn) -> UserOut:
    ...

# 常用选项
@app.get("/items/", response_model=list[Item],
         response_model_exclude_unset=True,     # 只返回显式设置过的字段
         response_model_exclude_none=True,
         response_model_exclude={"internal_id"})
```

> **面试落点**：「怎么防止密码/内部字段泄漏到 API 响应？」
> 答：**入参模型和出参模型分离**，用 `response_model` 强制过滤。
> 这是 FastAPI 项目的标准分层：`XxxCreate` / `XxxUpdate` / `XxxPublic` / `XxxInDB`。

## 4. `async def` vs `def`（最重要的一条）

```python
@app.get("/a")
async def a():                  # 直接在【事件循环】里运行
    await async_db.fetch()      # 里面必须全是异步调用

@app.get("/b")
def b():                        # FastAPI 自动放进【线程池】运行 ★
    return sync_db.query()      # 同步阻塞库放这里是安全的
```

**决策规则**：

| 你的代码 | 用什么 |
|---|---|
| 全是 `await` 的异步库（asyncpg、httpx、redis.asyncio） | `async def` |
| 用了同步阻塞库（psycopg2、requests、pandas、PIL） | **`def`**（让 FastAPI 丢线程池） |
| 纯 CPU 计算 | `def`，或显式丢进程池 |
| 混合 | `async def` + `await asyncio.to_thread(sync_part)` |

**最危险的错误**：

```python
@app.get("/danger")
async def danger():             # ❌ async def
    return requests.get(url)    # ❌ 里面是同步阻塞调用
    # → 阻塞整个事件循环，该 worker 上所有并发请求全部停滞
```

> **面试落点**：这是 FastAPI 面试的**必考题**。
> 「在 `async def` 路由里调用同步阻塞代码会怎样？」
> 答：阻塞事件循环，**整个 worker 的所有请求都被卡住**，而不只是当前请求；
> 正确做法是要么改用 `def`（FastAPI 会自动放线程池），要么 `await asyncio.to_thread(...)`。
> 补一句「线程池默认大小是 40（Starlette 的 `anyio` 默认 40 个 token）」更显专业。

```python
# 调整线程池大小
import anyio
limiter = anyio.to_thread.current_default_thread_limiter()
limiter.total_tokens = 100
```

> ⚠️ **别把两个线程池搞混**（面试追问时容易翻车）：
>
> | 谁的池 | 默认大小 | 触发方式 |
> |---|---|---|
> | **Starlette/FastAPI 跑 `def` 路由**的池 | **40**（anyio 的 token 限流器） | 路由写成 `def` |
> | **asyncio 的默认 executor** | **`min(32, cpu_count + 4)`** | `asyncio.to_thread()` / `run_in_executor(None, ...)` |
>
> 两者是**独立的两个池**，各自会排队。高并发下大量同步调用时，
> 两边都可能成为瓶颈，要分别调整。

**注意线程池救不了纯 Python 的 CPU 计算**——线程里照样争抢 GIL，
实测事件循环延迟仍有 20 倍以上劣化。那种情况必须上进程池，
完整方案见 [[web/fastapi-cpu-bound]]。

## 5. 请求与响应对象

```python
from fastapi import Request, Response, status
from fastapi.responses import JSONResponse, StreamingResponse, FileResponse, ORJSONResponse

@app.get("/raw")
async def raw(request: Request):
    request.method / request.url / request.headers / request.client.host
    request.query_params / request.path_params
    body = await request.body()
    data = await request.json()
    request.state.user_id        # 中间件塞进来的自定义状态

# 自定义响应
return JSONResponse(content={"a": 1}, status_code=201, headers={"X-Foo": "bar"})
return ORJSONResponse(data)                       # 更快的 JSON（需装 orjson）
return FileResponse("report.pdf", filename="r.pdf")
return StreamingResponse(gen(), media_type="text/event-stream")   # SSE

# 直接改 response（设 cookie / header 但仍返回模型）
@app.get("/x")
async def x(response: Response):
    response.set_cookie("k", "v", httponly=True, samesite="lax")
    response.headers["X-Total"] = "100"
    return {"ok": True}
```

**流式响应**（大文件、SSE、LLM token 流）：

```python
@app.get("/stream")
async def stream():
    async def gen():
        for i in range(100):
            yield f"data: {i}\n\n"
            await asyncio.sleep(0.1)
    return StreamingResponse(gen(), media_type="text/event-stream")
```

## 6. 状态码、错误与 CORS

```python
from fastapi import HTTPException, status

raise HTTPException(status_code=404, detail="Item not found")
raise HTTPException(
    status_code=status.HTTP_401_UNAUTHORIZED,
    detail="Invalid token",
    headers={"WWW-Authenticate": "Bearer"},
)

# 全局异常处理（业务异常 → HTTP 响应的映射）
@app.exception_handler(NotFoundError)
async def not_found(request: Request, exc: NotFoundError):
    return JSONResponse(status_code=404, content={"code": exc.code, "msg": str(exc)})

# 覆盖校验错误的格式
from fastapi.exceptions import RequestValidationError
@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    return JSONResponse(status_code=422, content={"errors": exc.errors()})
```

CORS（前端工程师最熟悉的部分）：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # ❌ 生产别用 ["*"] 配 credentials
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Total-Count"],
    max_age=600,
)
```

## 7. 路由组织

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["users"],
    dependencies=[Depends(verify_token)],        # 整组共用的依赖
    responses={404: {"description": "Not found"}},
)

@router.get("/{uid}")
async def get_user(uid: int): ...

app.include_router(router, prefix="/api/v1")
```

见 [[web/fastapi-architecture]] 的分层项目结构。

## 8. OpenAPI 自动化的价值（对前端团队）

```python
app = FastAPI(
    title="订单服务", version="2.0", description="…",
    openapi_tags=[{"name": "orders", "description": "订单相关"}],
    docs_url="/docs", redoc_url=None,           # 生产可关闭
    openapi_url=None,                            # 完全关闭 schema 暴露
)

@app.get("/items/{id}",
    summary="获取单个商品",
    description="详细说明…",
    response_description="商品详情",
    tags=["items"],
    deprecated=False,
    responses={404: {"model": ErrorOut}},
)
async def get_item(id: int): ...
```

**杀手锏**：`openapi.json` 可以直接生成 TypeScript 客户端：

```bash
npx openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts
# 或 openapi-generator / orval / hey-api
```

后端改了模型，前端类型自动更新——**这是你作为前端工程师最该在面试里强调的落地价值**。

## 9. 表单、文件与 WebSocket

```python
from fastapi import Form, File, UploadFile, WebSocket

@app.post("/login")
async def login(username: str = Form(), password: str = Form()): ...

@app.post("/upload")
async def upload(file: UploadFile = File()):
    content = await file.read()      # 或 file.file 拿到 SpooledTemporaryFile
    file.filename, file.content_type
    # 大文件要分块，别一次 read()

@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"echo: {data}")
    except WebSocketDisconnect:
        ...
```

## 相关

- [[web/fastapi-di]] —— 依赖注入系统
- [[web/pydantic]] —— 数据校验与序列化
- [[web/fastapi-architecture]] —— 项目分层与中间件
- [[web/fastapi-async-db]] —— 数据库集成
- [[web/wsgi-asgi]] —— 底层协议
- [[concurrency/asyncio-fundamentals]] —— `async def` 的原理
- [[language/typing]] —— 注解驱动的基础
- [[interview/question-bank-web]] —— FastAPI 面试题
