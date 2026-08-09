---
title: "代码质量工具链"
date: 2026-08-07
tags: [ruff, mypy, black, pre-commit, CI, 代码规范, PEP8]
sources: ["pep-0008-style-guide.rst"]
---

# 代码质量工具链

前端有 ESLint + Prettier + tsc + husky。Python 的对应物在 2024 年后被 **ruff** 大幅简化。

## 1. 工具对照

| 前端 | Python（2026 推荐） | 说明 |
|---|---|---|
| ESLint | **ruff check** | Rust 写的，比 flake8 快 100 倍，且内置了 flake8 + isort + pyupgrade + bugbear 等 50+ 插件的规则 |
| Prettier | **ruff format** | 兼容 black 的风格 |
| tsc | **mypy** 或 **pyright** | 静态类型检查 |
| husky + lint-staged | **pre-commit** | Git 钩子管理 |
| npm audit | **pip-audit** | 依赖漏洞 |
| — | **bandit** | 安全静态分析（ruff 的 `S` 规则已覆盖大部分） |

**被 ruff 取代的旧工具**：flake8、isort、pylint（部分）、pyupgrade、autoflake、
pydocstyle、black（ruff format 兼容）。**新项目直接上 ruff，不要再装那一堆。**

## 2. ruff 配置

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]
exclude = ["migrations", ".venv"]

[tool.ruff.lint]
select = [
    "E", "W",    # pycodestyle（PEP 8）
    "F",         # pyflakes（未使用变量、未定义名字）
    "I",         # isort（import 排序）
    "N",         # pep8-naming
    "UP",        # pyupgrade（自动用新语法：Optional[X] → X | None）★
    "B",         # flake8-bugbear（可变默认值、循环里的 lambda 等真 bug）★
    "SIM",       # flake8-simplify
    "C4",        # comprehensions
    "RUF",       # ruff 自有规则
    "ASYNC",     # flake8-async（asyncio 里的阻塞调用）★ 对 FastAPI 很有价值
    "S",         # bandit 安全规则
    "PTH",       # 用 pathlib 代替 os.path
    "TID",       # 禁止相对导入等
]
ignore = ["E501"]           # 行长交给 formatter

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]        # 测试里允许 assert
"__init__.py" = ["F401"]    # 允许 re-export

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.ruff.format]
quote-style = "double"
```

```bash
ruff check .                  # 检查
ruff check --fix .            # 自动修复
ruff check --watch .
ruff format .                 # 格式化
ruff format --check .         # CI 里只检查不改
```

> **强烈推荐开启 `B`（bugbear）和 `ASYNC`**——
> `B006` 直接抓住**可变默认参数**这个经典 bug，`ASYNC` 能抓住
> **`async def` 里调用同步阻塞函数**这个 FastAPI 头号事故源。
> 面试里说"我们用 ruff 的 ASYNC 规则在 CI 里拦阻塞调用"很有说服力。

## 3. mypy / pyright

```toml
[tool.mypy]
python_version = "3.12"
strict = true                       # 新项目直接开
warn_unreachable = true
warn_return_any = true
show_error_codes = true
plugins = ["pydantic.mypy"]

[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

```bash
mypy src/
pyright src/                  # 更快，VS Code 的 Pylance 就是它
```

**渐进式引入类型（老项目的正确做法）**：

```toml
[tool.mypy]
strict = false                     # 全局宽松

[[tool.mypy.overrides]]            # 新模块严格
module = ["app.services.*", "app.api.*"]
strict = true
```

> 前端类比：完全对应 TS 的 `strict: false` + 逐目录收紧。
> `# type: ignore[code]` ≈ `@ts-expect-error`。

## 4. pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key          # ★ 防止密钥进仓库

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, types-requests]
```

```bash
pre-commit install                 # 安装 git hook
pre-commit run --all-files         # 首次全量跑
pre-commit autoupdate
```

## 5. CI 流水线

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: {POSTGRES_PASSWORD: test, POSTGRES_DB: test}
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-retries 5
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: {enable-cache: true}
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy src/
      - run: uv run pytest --cov=src --cov-fail-under=80
      - run: uv run pip-audit
```

## 6. PEP 8 要点（会被问）

```python
# 命名
module_name.py          # 小写下划线
ClassName               # 大驼峰
function_name           # 小写下划线（不是 camelCase！）
CONSTANT_NAME           # 全大写
_internal               # 单下划线：内部
__mangled               # 双下划线：名字改写
_                       # 占位（丢弃的值）

# 缩进：4 个空格（不用 tab）
# 行长：79（PEP 8 原文）/ 88（black 默认）/ 100~120（多数团队实际）
# 导入顺序：标准库 → 第三方 → 本地，各组之间空一行
import os                        # 标准库
import sys

import fastapi                   # 第三方
from sqlalchemy import select

from app.models import User      # 本地

# 空行：顶层定义之间 2 行，类内方法之间 1 行
# 引号：PEP 8 不规定，团队统一即可（black/ruff 默认双引号）
# 比较
if x is None:            # ✅ 不是 == None
if not items:            # ✅ 不是 len(items) == 0
if isinstance(x, int):   # ✅ 不是 type(x) == int
```

> **面试落点**：被问 PEP 8 时，除了背规则，更该说
> **「我们不靠人记规则，靠 ruff format + ruff check 在 pre-commit 和 CI 里强制」**——
> 这才是工程化的回答。

## 7. 文档字符串

```python
def transfer(src: Account, dst: Account, amount: Decimal) -> Transaction:
    """在两个账户间转账。

    Args:
        src: 转出账户，余额必须充足。
        dst: 转入账户。
        amount: 转账金额，必须为正。

    Returns:
        生成的交易记录。

    Raises:
        InsufficientFunds: 余额不足时。
        ValueError: amount <= 0 时。
    """
```

风格：**Google 风格**（上例，最流行）、NumPy 风格（科学计算）、reST（Sphinx 原生）。
工具：`mkdocs-material` + `mkdocstrings`（现代）或 Sphinx（传统）。

## 8. 代码质量的实际标准

一个成熟 Python 项目的 checklist：

```text
□ pyproject.toml 单一配置入口
□ ruff check + ruff format 在 pre-commit 和 CI 里强制
□ mypy strict（至少对核心模块）
□ pytest 覆盖率门槛（分支覆盖 ≥ 70-80%）
□ 锁文件提交进仓库（uv.lock）
□ pip-audit / trivy 扫描
□ src layout + 显式 __all__
□ 类型注解覆盖公开 API
□ 结构化日志，不用 print
□ 没有裸 except、没有可变默认参数、没有 shell=True 拼接
□ 依赖分组：运行时 / 开发 / 可选
□ CHANGELOG + 语义化版本
```

## 相关

- [[engineering/packaging-envs]] —— pyproject.toml 配置
- [[engineering/testing]] —— pytest 配置
- [[language/typing]] —— 类型注解基础
- [[web/fastapi-production]] —— 生产环境的质量要求
- [[bridge/npm-vs-pip]] —— 与前端工具链的完整对照
- [[sources/cpython-docs]] —— 来源：PEP 8
