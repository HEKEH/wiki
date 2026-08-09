---
title: "标准库必备模块速查"
date: 2026-08-07
tags: [标准库, pathlib, datetime, json, logging, subprocess, argparse]
sources: ["python-cheatsheet.md"]
---

# 标准库必备模块速查

Python 的卖点是「self-batteries included」。这一页覆盖**后端工程日常 90% 会用到**的标准库，
按"前端有什么 / Python 用什么"组织。

## 1. pathlib —— 路径操作（别再用 os.path）

```python
from pathlib import Path

p = Path("/var/log") / "app" / "x.log"      # / 运算符拼接，跨平台
p.name          #=> 'x.log'
p.stem          #=> 'x'
p.suffix        #=> '.log'
p.parent        #=> Path('/var/log/app')
p.parents[1]    #=> Path('/var/log')
p.absolute() / p.resolve()                  # resolve 会解析符号链接
p.exists() / p.is_file() / p.is_dir()
p.stat().st_size / st_mtime

p.read_text(encoding="utf-8")               # 一行读文件
p.write_text(content, encoding="utf-8")
p.read_bytes() / p.write_bytes(b)

p.parent.mkdir(parents=True, exist_ok=True) # mkdir -p
p.unlink(missing_ok=True)                   # rm -f
p.rename(target) / p.replace(target)

Path.cwd() / Path.home()
Path(__file__).parent                       # 当前脚本目录 —— 最常用！
list(Path("src").glob("**/*.py"))           # 递归 glob
list(Path("src").rglob("*.py"))             # 同上，更短
Path("a/b").with_suffix(".json")
```

> 项目根目录的标准写法：`ROOT = Path(__file__).resolve().parents[2]`

## 2. datetime —— 时间处理

```python
from datetime import datetime, date, time, timedelta, timezone, UTC
from zoneinfo import ZoneInfo          # 3.9+ 标准库时区（替代 pytz）

datetime.now()                          # ❌ naive（无时区），后端代码禁用
datetime.now(UTC)                       # ✅ 3.11+  aware
datetime.now(timezone.utc)              # ✅ 通用写法
datetime.now(ZoneInfo("Asia/Shanghai"))

dt = datetime(2026, 8, 7, 12, 0, tzinfo=UTC)
dt.isoformat()                          #=> '2026-08-07T12:00:00+00:00'
datetime.fromisoformat("2026-08-07T12:00:00+00:00")    # 3.11 起支持完整 ISO 8601
dt.strftime("%Y-%m-%d %H:%M:%S")
datetime.strptime("2026-08-07", "%Y-%m-%d")
dt.timestamp() / datetime.fromtimestamp(ts, UTC)

dt + timedelta(days=1, hours=3)
(dt2 - dt1).total_seconds()
dt.astimezone(ZoneInfo("America/New_York"))
```

**铁律（后端面试会问）**：

1. **存储和传输一律用 UTC**，只在展示层转本地时区。
2. **永远用 aware datetime**，naive 和 aware 相减会 `TypeError`。
3. 数据库列用 `TIMESTAMP WITH TIME ZONE`。
4. 计时用 `time.monotonic()` / `time.perf_counter()`，**不要用 `time.time()`**
   （系统时钟可能被 NTP 调整甚至回拨）。

```python
import time
time.time()             # Unix 时间戳（墙钟，可能回拨）
time.monotonic()        # 单调时钟，用于超时判断
time.perf_counter()     # 最高精度，用于性能计时
time.sleep(0.5)
```

## 3. json

```python
import json

json.dumps(obj, ensure_ascii=False, indent=2, default=str, sort_keys=True)
json.loads(s)
json.dump(obj, fp) / json.load(fp)

# 自定义序列化
class Enc(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, datetime): return o.isoformat()
        if isinstance(o, Decimal):  return str(o)
        return super().default(o)
json.dumps(obj, cls=Enc)
```

**必知的坑**：

```python
json.dumps({"中": 1})                       #=> '{"\\u4e2d": 1}'    默认转义非 ASCII
json.dumps({"中": 1}, ensure_ascii=False)   #=> '{"中": 1}'         ✅ 通常要加
json.dumps({1: "a"})                        #=> '{"1": "a"}'  int 键被转成字符串！
json.dumps(float("nan"))                    #=> 'NaN'   非法 JSON，跨语言会炸
                                            #   用 allow_nan=False 强制报错
json.dumps(Decimal("0.1"))                  # ❌ TypeError
```

性能：`orjson`（Rust）比标准库快 5~10 倍，FastAPI 可用 `ORJSONResponse`。

## 4. logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s [%(name)s] %(message)s",
)
log = logging.getLogger(__name__)     # ✅ 每个模块一个 logger，用 __name__

log.debug("详细: %s", x)               # ✅ 惰性格式化
log.info("...")
log.warning("...")
log.error("...")
log.exception("失败")                  # 在 except 块里用，自动附加 traceback
log.critical("...")
```

要点：

- **库代码不要 `basicConfig`**，只 `getLogger(__name__)`，配置留给应用。
- **不要用 f-string 传消息**（见 [[language/strings-encoding]]）。
- 生产用**结构化日志**（`structlog` / `python-json-logger`），便于 ELK/Loki 检索。
- Logger 有层级（`a.b.c` 继承 `a.b` 的配置），`propagate=False` 可断开。

```python
# 结构化日志示例
import structlog
log = structlog.get_logger()
log.info("order_created", order_id=123, user_id=456, amount=99.9)
```

## 5. os / sys / subprocess / shutil

```python
import os, sys, shutil, subprocess

os.environ.get("DATABASE_URL", "sqlite:///dev.db")
os.getenv("PORT", "8000")
os.cpu_count()
os.getpid()

sys.argv / sys.exit(1) / sys.stdin / sys.stdout / sys.stderr
sys.platform / sys.version_info >= (3, 12)

shutil.copy2(src, dst) / copytree / rmtree / move / which("git") / disk_usage("/")

# subprocess —— 永远用列表形式，永远不要 shell=True 拼接用户输入
r = subprocess.run(
    ["git", "rev-parse", "HEAD"],
    capture_output=True, text=True, check=True, timeout=10, cwd=repo,
)
r.stdout.strip()

# ❌ 命令注入风险
subprocess.run(f"git log {user_input}", shell=True)
```

> **安全考点**：`shell=True` + 用户输入 = 命令注入。面试问"Python 常见安全问题"时，
> 除了它还应答出：`eval`/`exec`/`pickle.loads` 处理不可信数据（**pickle 可以执行任意代码**）、
> SQL 字符串拼接、YAML 用 `yaml.load` 而非 `safe_load`。

## 6. argparse —— CLI

```python
import argparse

p = argparse.ArgumentParser(description="工具说明")
p.add_argument("input", help="输入文件")                     # 位置参数
p.add_argument("-o", "--output", default="out.txt")
p.add_argument("-v", "--verbose", action="store_true")
p.add_argument("-n", type=int, default=10)
p.add_argument("--mode", choices=["fast", "safe"])
p.add_argument("--tags", nargs="*")
sub = p.add_subparsers(dest="cmd")                           # 子命令
args = p.parse_args()
```

现代替代：**Typer**（FastAPI 作者出品，基于类型注解）、`click`。

```python
import typer
app = typer.Typer()

@app.command()
def build(src: str, out: str = "dist", verbose: bool = False):
    ...
app()      # 注解自动变成 CLI 参数和帮助
```

## 7. 其它高频模块

```python
# 数值
import math, decimal, fractions, statistics, random, secrets
decimal.Decimal("0.1") + Decimal("0.2")    #=> Decimal('0.3')  ★ 金额必须用它
0.1 + 0.2                                   #=> 0.30000000000000004  浮点误差
random.random() / randint / choice / shuffle / sample
secrets.token_urlsafe(32)                   # ★ 密码学安全，token/密钥必须用它而非 random

# 文本
import re, textwrap, difflib, string, unicodedata
textwrap.dedent(sql)                        # 去掉多行字符串的公共缩进

# 数据
import csv, sqlite3, pickle, base64, hashlib, hmac, uuid, zlib, gzip
hashlib.sha256(b"x").hexdigest()
hmac.compare_digest(a, b)                   # ★ 恒定时间比较，防时序攻击
uuid.uuid4()

# 系统与并发
import threading, multiprocessing, asyncio, queue, signal, atexit
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# 开发
import typing, dataclasses, enum, abc, inspect, warnings, traceback
import unittest, doctest, pdb, timeit, cProfile, tracemalloc
import functools, itertools, collections, contextlib, copy, operator

# 网络
import socket, http, urllib.request, ipaddress, email
# 实践中 HTTP 用第三方：requests（同步）/ httpx（同步+异步）/ aiohttp
```

## 8. 与 Node 标准库对照

| Node | Python |
|---|---|
| `fs` / `fs.promises` | `pathlib` / `open()` / `aiofiles` |
| `path` | `pathlib` |
| `os` | `os` / `platform` / `sys` |
| `child_process` | `subprocess` |
| `crypto` | `hashlib` / `hmac` / `secrets` / `cryptography`(第三方) |
| `util.promisify` | `asyncio.to_thread` |
| `events` (EventEmitter) | 无内置；用回调 / `blinker` / `pyee` |
| `stream` | 迭代器 / 生成器 / `io` |
| `worker_threads` | `threading`（受 GIL）/ `multiprocessing` |
| `cluster` | `multiprocessing` / gunicorn workers |
| `http` | `http.server`（仅测试）/ FastAPI + uvicorn |
| `JSON` | `json` |
| `console.log` | `print` / `logging` |
| `process.env` | `os.environ` |
| `process.argv` | `sys.argv` / `argparse` |
| `assert` | `assert`（会被 -O 移除）/ `unittest` |
| `Date` | `datetime` + `zoneinfo` |
| `Intl.NumberFormat` | f-string 的 `{n:,}` / `babel`(第三方) |

## 相关

- [[stdlib/collections-itertools]] —— 容器与迭代工具
- [[stdlib/builtin-data-structures]] —— 内置类型
- [[language/strings-encoding]] —— 编码与格式化
- [[engineering/packaging-envs]] —— 第三方库管理
- [[web/fastapi-production]] —— 日志与可观测性落地
