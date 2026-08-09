---
title: "包管理、虚拟环境与打包"
date: 2026-08-07
tags: [uv, pip, venv, pyproject, poetry, 依赖锁, 打包]
sources: ["fastapi-doc/virtual-environments.md"]
---

# 包管理、虚拟环境与打包

这是前端工程师转 Python **最不适应**的一块：npm 一统天下，而 Python 的工具链
经历了 20 年的碎片化。好消息是 **2024 年起 `uv` 正在统一这一切**。

## 1. 为什么必须用虚拟环境

Python **没有 node_modules**——默认所有包装在**全局的 site-packages** 里。
两个项目要不同版本的同一个库就会冲突。

```bash
python -m venv .venv                  # 创建（标准库自带）
source .venv/bin/activate             # 激活（Windows: .venv\Scripts\activate）
deactivate

which python                          # 确认指向 .venv/bin/python
python -c "import sys; print(sys.prefix)"
```

> 前端类比：**venv ≈ 强制的 node_modules**，只不过要手动创建和激活。
> 忘记激活就 `pip install` 会污染全局环境——这是新手最常见的事故。
> `uv` 会自动处理这一步，是它最大的体验改进之一。

## 2. `uv` —— 2026 年的推荐方案 ★

Astral（ruff 的作者）用 Rust 写的一体化工具，**比 pip 快 10~100 倍**，
一个工具替代 pip + venv + pyenv + pip-tools + pipx + poetry。

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh

# 项目初始化
uv init myapp && cd myapp             # 生成 pyproject.toml
uv python install 3.12                # 安装 Python 本身（替代 pyenv）
uv venv --python 3.12                 # 创建虚拟环境

# 依赖管理
uv add fastapi "uvicorn[standard]"    # 加依赖（自动更新 pyproject + uv.lock + 安装）
uv add --dev pytest ruff mypy         # 开发依赖
uv remove requests
uv sync                                # 按 uv.lock 精确还原环境（CI 用这个）
uv sync --frozen --no-dev              # 生产：不改 lock，不装 dev 依赖
uv lock --upgrade-package fastapi      # 升级单个包

# 运行（自动激活环境，无需 source）
uv run python main.py
uv run pytest
uv run uvicorn app.main:app --reload

# 工具（替代 pipx）
uvx ruff check .                       # 临时运行，不污染项目
uv tool install ruff
```

| npm/pnpm | uv |
|---|---|
| `npm init` | `uv init` |
| `npm install` | `uv sync` |
| `npm install express` | `uv add fastapi` |
| `npm install -D vitest` | `uv add --dev pytest` |
| `npm run dev` | `uv run ...` |
| `npx cowsay` | `uvx cowsay` |
| `package.json` | `pyproject.toml` |
| `package-lock.json` | **`uv.lock`** |
| `node_modules/` | `.venv/` |
| `nvm` | `uv python install` |

## 3. `pyproject.toml` —— 单一配置文件

```toml
[project]
name = "myapp"
version = "0.1.0"
description = "A FastAPI service"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "pydantic-settings>=2.0",
]

[project.optional-dependencies]        # 可选特性组：pip install myapp[redis]
redis = ["redis>=5.0"]

[dependency-groups]                    # PEP 735，开发依赖（uv 原生支持）
dev = ["pytest>=8", "pytest-asyncio", "httpx", "ruff", "mypy"]

[project.scripts]
myapp = "app.cli:main"                 # 装完后可直接敲 myapp 命令

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/app"]

# 所有工具的配置都集中在这里（≈ 前端的 package.json + eslintrc + tsconfig 合一）
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF"]

[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

> **面试落点**：`pyproject.toml`（PEP 518/517/621）是**现代 Python 项目的唯一配置入口**，
> 取代了 `setup.py` + `setup.cfg` + `requirements.txt` + 各种 `.ini`。
> 看到候选人还在用 `setup.py` 和裸 `requirements.txt` 会觉得知识陈旧。

## 4. 版本约束与依赖锁

```toml
"fastapi==0.115.0"      # 精确锁定
"fastapi>=0.115,<0.116" # 范围
"fastapi~=0.115.0"      # 兼容版本：>=0.115.0, ==0.115.*（≈ npm 的 ~）
"fastapi>=0.115"        # 最低版本（库最常用）
```

**核心区别（面试常问）**：

| | `pyproject.toml` 的 dependencies | `uv.lock` / `requirements.txt` |
|---|---|---|
| 内容 | **抽象**依赖（范围） | **具体**版本 + 哈希 + 全部传递依赖 |
| 谁用 | 库、应用都写 | **应用**必须提交进仓库 |
| 类比 | `package.json` 的 dependencies | `package-lock.json` |

```bash
# 生成给不支持 uv 的环境用的 requirements.txt
uv pip compile pyproject.toml -o requirements.txt          # 带哈希，可复现
uv export --format requirements-txt > requirements.txt
```

> **原则**：**库（要被别人 import）写宽松范围；应用（要部署）提交锁文件**。
> 这跟前端一模一样。

## 5. 其它工具（要认识）

| 工具 | 定位 | 现状 |
|---|---|---|
| **uv** | 一体化，Rust | **2026 年首选** ✅ |
| **pip** | 官方安装器 | 永远可用的底线；`pip install -e .` 可编辑安装 |
| **venv** | 官方虚拟环境 | 标准库自带 |
| **poetry** | 依赖管理 + 打包 | 成熟、生态大，但比 uv 慢很多 |
| **pdm** | 类似 poetry，PEP 标准优先 | 小众但设计好 |
| **pipenv** | 早期方案 | 基本被淘汰 |
| **conda / mamba** | 科学计算，管理非 Python 依赖（CUDA/MKL） | 数据/AI 领域仍是主流 |
| **pyenv** | 多版本 Python | 被 `uv python` 取代中 |
| **pipx** | 全局安装 CLI 工具 | 被 `uv tool` 取代中 |
| **hatch / setuptools / flit** | 构建后端 | pyproject 的 `build-backend` |

```bash
# 可编辑安装（开发时让改动立即生效，≈ npm link）
uv pip install -e .
pip install -e ".[dev]"
```

## 6. 发布一个包

```bash
uv build                       # 生成 dist/*.whl 和 dist/*.tar.gz
uv publish                     # 上传到 PyPI（或 twine upload dist/*）
```

两种分发格式：

- **wheel（`.whl`）** —— 预编译的二进制分发，**安装快、无需编译**。
  文件名编码了平台：`myapp-1.0-cp312-cp312-manylinux_x86_64.whl`。
- **sdist（`.tar.gz`）** —— 源码分发，安装时需要构建。

> **面试落点**：「wheel 和 sdist 的区别？」——wheel 是预编译产物，
> 直接解压即可用；sdist 是源码，装的时候要跑构建（有 C 扩展时需要编译器）。
> 这就是 `pip install numpy` 通常很快（有 wheel）而某些包很慢（要现场编译）的原因。

## 7. Docker 中的依赖安装

```dockerfile
# ✅ 分层缓存：依赖变了才重装
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY src/ ./src/
RUN uv sync --frozen --no-dev

# ❌ 一次 COPY 全部 → 改一行代码就重装所有依赖
COPY . .
RUN uv sync
```

## 8. 常见问题速查

| 问题 | 原因 / 解法 |
|---|---|
| `ModuleNotFoundError` 但明明装了 | 装到了别的环境；`which python` / `pip -V` 确认路径 |
| `pip install` 报权限错误 | 没激活 venv，在往系统 Python 装；**永远别 `sudo pip`** |
| 依赖冲突（A 要 x<2，B 要 x>=2） | uv/poetry 会报解析失败；升级或用 optional-dependencies 隔离 |
| 本地能跑线上不行 | 没有锁文件；CI 用 `uv sync --frozen` |
| 导入自己的包失败 | 用 src layout + `pip install -e .`，见 [[language/modules-imports]] |
| 包名与模块名不同 | `pip install PyYAML` → `import yaml`；`pip install psycopg2-binary` → `import psycopg2` |
| 安装超慢 | 换镜像源：`uv add --index-url https://pypi.tuna.tsinghua.edu.cn/simple ...` |

```bash
# 国内镜像永久配置
export UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

## 9. 安全

```bash
pip-audit                      # 扫描已知漏洞（≈ npm audit）
uv pip list --outdated
trivy image myapp:latest       # 镜像扫描
```

**供应链风险**：PyPI 上存在**typosquatting**（`reqeusts` vs `requests`）
和恶意包。`pip install` 会执行 `setup.py`——**安装本身就能执行任意代码**。
生产环境应锁定版本 + 校验哈希 + 用私有镜像源。

## 相关

- [[language/modules-imports]] —— 导入系统与 src layout
- [[engineering/tooling-quality]] —— ruff/mypy 的 pyproject 配置
- [[web/fastapi-production]] —— Docker 中的依赖安装
- [[bridge/npm-vs-pip]] —— 与 npm 生态的完整对照
- [[interview/question-bank-web]] —— 工程化面试题
