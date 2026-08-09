---
title: "FastAPI 生产部署与性能"
date: 2026-08-07
tags: [部署, uvicorn, gunicorn, docker, 可观测性, 性能调优]
sources: ["fastapi-doc/deployment/concepts.md", "fastapi-doc/deployment/docker.md"]
---

# FastAPI 生产部署与性能

面试到最后往往会问：**「你的服务怎么部署？怎么定容量？出问题怎么排查？」**
这一页是标准答案。

## 1. 并发模型：worker 与事件循环的关系

```text
一台机器
├── 进程 1（worker）──> 1 个事件循环 ──> N 个并发协程（连接）
├── 进程 2（worker）──> 1 个事件循环 ──> N 个并发协程
└── ...
```

**关键认知**：

- **每个 worker 进程一个事件循环**（asyncio 单线程）。
- **一个 worker 能处理的并发连接数取决于协程数量，不是线程数**——I/O 密集时轻松上千。
- **CPU 并行度 = worker 数**（受 GIL 限制，见 [[internals/gil]]）。
- **worker 数的经验值：`CPU 核数`（纯异步 I/O）到 `2 × 核数 + 1`（混合负载）**。
  worker 越多内存占用越高（每个都是完整的 Python 进程）。

```bash
# ① 单进程（容器化推荐，副本数交给 K8s）
uvicorn app.main:app --host 0.0.0.0 --port 8000

# ② uvicorn 自带多 worker
uvicorn app.main:app --workers 4

# ③ gunicorn 管进程 + uvicorn 处理协议（传统 VM 部署）
gunicorn app.main:app \
  -k uvicorn.workers.UvicornWorker -w 4 \
  --bind 0.0.0.0:8000 \
  --timeout 60 --graceful-timeout 30 \
  --max-requests 1000 --max-requests-jitter 100 \
  --access-logfile - --error-logfile -

# ④ FastAPI CLI（0.111+）
fastapi run app/main.py --workers 4
```

> **面试落点**：「你的服务开几个 worker？为什么？」
> 答：**先看负载类型**。纯异步 I/O 密集时 worker 数 ≈ CPU 核数就够（并发靠协程不靠进程）；
> 如果路由里有同步阻塞代码（跑在线程池里）或 CPU 计算，就要考虑 `2×核数+1` 并做压测确认。
> **容器环境下我倾向单容器单 worker，把副本伸缩交给 K8s HPA**——这样资源限制、
> 健康检查、优雅关闭都更可控，也避免了容器内进程管理的复杂性。

`--max-requests` 的作用：**定期重启 worker 以释放内存**（见 [[internals/memory-model]] 里
"内存不还给 OS"的问题）——这是 Python 服务的常规实践，不是心虚。

## 2. Dockerfile（多阶段 + uv）

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project

COPY src/ ./src/
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev

# ── 运行阶段 ──
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PATH="/app/.venv/bin:$PATH"

RUN useradd -m -u 1000 app                 # ★ 非 root 运行
WORKDIR /app
COPY --from=builder --chown=app:app /app /app
USER app

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

要点清单：

- `PYTHONUNBUFFERED=1` —— 否则日志被缓冲，容器里看不到实时输出。
- `PYTHONDONTWRITEBYTECODE=1` —— 不生成 `.pyc`，镜像更干净。
- **依赖层与代码层分开 COPY** —— 改代码不会让依赖层缓存失效。
- **非 root 用户**、`slim` 基础镜像、多阶段构建。
- **不要用 `--reload`**（生产），不要把 `.env` 打进镜像。

## 3. 优雅关闭（K8s 必备）

```text
K8s 发 SIGTERM
  → uvicorn 停止接受新连接
  → 等待进行中的请求完成（graceful timeout）
  → 触发 lifespan 的关闭段（关连接池、刷缓冲）
  → 进程退出
超时未退出 → SIGKILL（强杀，可能丢请求）
```

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    # ⚠️ 这里的清理要快（几秒内），否则会被 SIGKILL
    await engine.dispose()
    await redis.aclose()
```

```yaml
# K8s
spec:
  terminationGracePeriodSeconds: 45      # 要大于 graceful-timeout
  containers:
    - lifecycle:
        preStop:
          exec: {command: ["sleep", "5"]}   # 等 Service 摘掉端点，避免流量还进来
      readinessProbe:
        httpGet: {path: /health, port: 8000}
      livenessProbe:
        httpGet: {path: /health, port: 8000}
        initialDelaySeconds: 10
```

> **面试落点**：优雅关闭里最容易被忽略的是 **preStop 的 sleep**——
> Pod 被删除时，"从 Service endpoints 摘除"和"发 SIGTERM"是**并行**的，
> 不等一下就会有流量打到正在关闭的 Pod 上。这个细节能体现真实运维经验。

## 4. 可观测性三件套

```python
# ① 结构化日志（JSON，便于检索）
import structlog
structlog.configure(processors=[
    structlog.contextvars.merge_contextvars,        # 自动带上 request_id
    structlog.processors.add_log_level,
    structlog.processors.TimeStamper(fmt="iso"),
    structlog.processors.JSONRenderer(),
])
log = structlog.get_logger()
log.info("order_created", order_id=1, user_id=2, amount=99.9)

# ② 指标（Prometheus）
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app, endpoint="/metrics")
# 关键指标：QPS、P50/P95/P99 延迟、错误率、进行中请求数、连接池使用率

# ③ 链路追踪（OpenTelemetry）
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
FastAPIInstrumentor.instrument_app(app)
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)
```

**RED 方法**（该监控什么）：**R**ate（QPS）、**E**rrors（错误率）、**D**uration（延迟分布）。
加上资源侧的 CPU / 内存 / 连接池 / 队列深度。

## 5. 性能调优清单

按收益排序：

```python
# ① 消灭阻塞事件循环的调用 —— 收益最大！
#    检查：async def 路由里有没有 requests / time.sleep / 同步 DB / 重 CPU
asyncio.run(main(), debug=True)          # 开发期开启慢回调警告
py-spy top --pid <pid>                    # 生产采样，看时间花在哪

# ② 数据库：N+1、缺索引、连接池
#    - selectinload/joinedload（见 fastapi-async-db）
#    - EXPLAIN ANALYZE 慢查询
#    - pool_size 与 worker 数的乘积不能超过 DB 的 max_connections！★
#      4 workers × pool_size 20 = 80 个连接。用 pgbouncer 做连接复用。

# ③ 缓存
from redis.asyncio import Redis
# 应用级：@cache 装饰（fastapi-cache2）
# HTTP 级：ETag / Cache-Control
# CDN 级：静态资源与可缓存 GET

# ④ 序列化
app = FastAPI(default_response_class=ORJSONResponse)   # orjson 快 2~5 倍
# 大列表响应考虑分页 + 只返回必要字段

# ⑤ 事件循环
pip install uvloop httptools           # uvicorn 会自动使用

# ⑥ Pydantic
# 热路径避免重复构建模型；用 model_validate_json 而非 json.loads+validate

# ⑦ 压缩
from fastapi.middleware.gzip import GZipMiddleware
app.add_middleware(GZipMiddleware, minimum_size=1000)
```

**压测**：

```bash
# wrk / hey / locust / k6
hey -n 10000 -c 100 http://localhost:8000/items/1
locust -f locustfile.py --headless -u 500 -r 50 -t 2m
```

> **面试落点**：「服务变慢了怎么排查？」标准流程：
> **① 看监控确认是延迟还是错误率、是全局还是某个端点；
> ② 看链路追踪定位是 DB / 外部调用 / 应用自身；
> ③ 应用自身慢就用 py-spy 采样看火焰图，重点查是不是阻塞了事件循环；
> ④ DB 慢就 EXPLAIN + 看慢查询日志 + 查 N+1；
> ⑤ 确认连接池/线程池有没有排队。**

## 6. 配置与密钥

```python
# ✅ 从环境变量注入，用 pydantic-settings 校验
class Settings(BaseSettings):
    database_url: str
    secret_key: SecretStr
    # 缺失时进程启动就失败 —— fail fast

# ❌ 密钥写进代码 / 提交进 git / 打进镜像
# ✅ K8s Secret / Vault / AWS Secrets Manager / SOPS
```

**12-factor 要点**：配置在环境里、日志到 stdout、进程无状态、优雅启停、
开发与生产环境尽量一致。

## 7. 部署形态对比

| 形态 | 适合 | 注意 |
|---|---|---|
| **K8s Deployment** | 中大型团队 | HPA 按 CPU/QPS 扩缩；单容器单 worker |
| **Docker Compose** | 小项目/自托管 | 前面加 nginx/Caddy 做 TLS |
| **Serverless**（Lambda + Mangum） | 低频/突发流量 | **冷启动**痛点；长连接/WebSocket 不友好 |
| **PaaS**（Railway/Fly.io/Render） | 快速上线 | 成本随规模上升 |
| 传统 VM + systemd + gunicorn | 遗留环境 | 手工运维成本高 |

```python
# Serverless 适配
from mangum import Mangum
handler = Mangum(app)      # AWS Lambda 入口
```

## 8. 上线前检查清单

```text
□ /health（liveness）与 /ready（readiness，检查 DB/Redis）分开
□ 生产关闭 /docs 和 /openapi.json（或加鉴权）
□ CORS 白名单，不用 "*" 搭配 credentials
□ 所有密钥来自环境变量，不在镜像/代码里
□ 结构化日志 + request_id 贯穿
□ Prometheus 指标 + 告警规则
□ 优雅关闭 + terminationGracePeriodSeconds + preStop
□ 资源 limits/requests（内存 limit 要留足，OOMKill 很难查）
□ DB 连接数 = worker 数 × pool_size ≤ DB max_connections
□ 数据库迁移向后兼容（滚动发布期间新旧版本共存）
□ 限流（登录、写接口）
□ 依赖锁文件（uv.lock / poetry.lock）提交进仓库
□ 镜像扫描（trivy）+ 依赖漏洞扫描（pip-audit）
□ 压测过并知道单实例的 QPS 上限
```

## 相关

- [[web/wsgi-asgi]] —— gunicorn 与 uvicorn 的关系
- [[web/fastapi-architecture]] —— lifespan 与优雅关闭
- [[web/fastapi-async-db]] —— 连接池配置
- [[internals/gil]] —— worker 数与 CPU 并行
- [[internals/memory-model]] —— 为什么要 max-requests
- [[engineering/performance]] —— 剖析工具
- [[engineering/packaging-envs]] —— uv 与依赖锁
- [[interview/question-bank-web]] —— 部署与性能面试题
