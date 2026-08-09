---
title: "模块、包与导入系统"
date: 2026-08-07
tags: [模块, 包, import, 循环导入, sys.path, __init__]
sources: ["python-cheatsheet.md", "cpython-doc/faq-programming.rst"]
---

# 模块、包与导入系统

前端有 ESM/CJS 两套模块系统，Python 只有一套，但它的**搜索路径 + 缓存 + 循环导入**行为
和 Node 差别很大，是转 Python 时踩坑最多的地方之一。

## 1. 模块与包

- **模块（module）**：一个 `.py` 文件。
- **包（package）**：含 `__init__.py` 的目录（`__init__.py` 可以是空文件）。
- **命名空间包**：无 `__init__.py` 的目录（PEP 420），可跨路径合并，一般不要主动用。

```text
myapp/
├── __init__.py          # 包的入口，import myapp 时执行
├── main.py
├── core/
│   ├── __init__.py
│   └── db.py
└── api/
    ├── __init__.py
    └── routes.py
```

## 2. import 的四种形式

```python
import os                          # 绑定名字 os
import os.path                     # 仍然只绑定 os（然后可以 os.path.join）
import numpy as np                 # 别名
from os import path, sep           # 只绑定 path、sep
from os.path import join as pjoin
from .core import db               # 相对导入（只能在包内用）
from ..utils import helper         # 上一级
from mod import *                  # ❌ 污染命名空间，只在 REPL 用
```

`from mod import *` 导入哪些名字：模块定义了 `__all__` 就按它，否则所有不以 `_` 开头的顶层名字。

```python
# mymodule.py
__all__ = ["public_api", "Config"]     # 显式声明公开 API（也是给 linter 看的）
```

### 绝对导入 vs 相对导入

```python
# ✅ 推荐：绝对导入，路径清晰、可移动
from myapp.core.db import get_session

# 相对导入：包内重构友好，但脚本直接运行会失败
from .core.db import get_session      # 只有作为包的一部分被导入时才有效
```

> **经典报错**：`ImportError: attempted relative import with no known parent package`
> ——原因是你用 `python myapp/main.py` 直接跑了含相对导入的文件。
> 正解：`python -m myapp.main`（以模块方式运行，`__package__` 才会被正确设置）。

## 3. 模块搜索路径

```python
import sys
sys.path
#=> ['', '/path/to/script/dir', '<zip>', '/usr/lib/python3.12', ..., 'site-packages']
```

顺序（先找到先用）：

1. **脚本所在目录**（或 `python -m` 时的当前工作目录）
2. `PYTHONPATH` 环境变量
3. 标准库
4. `site-packages`（第三方包）

> **头号新手事故**：把自己的文件命名为 `random.py` / `json.py` / `queue.py` / `types.py`，
> 它会**遮蔽标准库**，产生莫名其妙的错误。检查方法：`python -c "import json; print(json.__file__)"`。

## 4. 模块缓存：只执行一次

```python
import sys
sys.modules["json"]        # 所有已导入模块的缓存字典
```

**同一个模块无论被 import 多少次，模块体只执行一次**。这是单例模式最简单的实现方式：

```python
# config.py
settings = load_settings()     # 全进程只执行一次

# 别处
from config import settings    # 拿到的永远是同一个对象
```

重新加载（调试用，生产别用）：

```python
import importlib
importlib.reload(mod)
```

> 与 Node 的差别：Node 的 CJS 也有 `require.cache`，行为类似；
> 但 Python 缓存的 key 是**模块的全限定名**，同一个文件用不同的导入路径（`myapp.db` vs `db`）
> 会被当成**两个不同的模块**，各执行一遍——这会造成"单例出现两份"的诡异 bug。

## 5. `__name__ == "__main__"`

```python
def main(): ...

if __name__ == "__main__":     # 作为脚本运行时 __name__ == "__main__"
    main()                     # 被 import 时 __name__ == 模块名，不执行
```

**必须写**的三个理由：

1. 模块被 import 时不会意外执行。
2. `multiprocessing` 在 spawn 模式（macOS/Windows 默认）下**会重新导入主模块**，
   不加这个守卫会无限递归创建进程。
3. 测试框架导入你的模块时不会执行副作用。

## 6. 循环导入（面试高频）

```python
# a.py
from b import bee
def ay(): return bee()

# b.py
from a import ay          # ❌ ImportError: cannot import name 'ay' from partially
def bee(): return ay()    #    initialized module 'a' (most likely due to a circular import)
```

**原因**：导入 `a` 时，`a` 的模块对象已放进 `sys.modules` 但**只执行到第一行**，
此时 `b` 反过来 `from a import ay`，`a` 里还没定义 `ay`，于是失败。

**四种解法**：

```python
# ① 改用 import 模块而非 from ... import 名字（延迟到调用时才解析属性）
import b
def ay(): return b.bee()          # ✅ 此时 b 已完全初始化

# ② 把导入移进函数内部（函数体在调用时才执行）
def ay():
    from b import bee
    return bee()

# ③ 提取公共依赖到第三个模块（最正确的架构解法）
#    a → common ← b

# ④ 仅类型注解需要时，用 TYPE_CHECKING（运行时不导入）
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from b import Bee
def f(x: "Bee") -> None: ...
```

> **面试落点**：能说出「循环导入的本质是**模块只被部分初始化**，而 `from X import name`
> 要求 name 此刻已存在；`import X` 只要求模块对象存在」——这一句就够了。
> 顺带说出「循环导入通常是架构分层出了问题，应该抽公共模块」是加分项。

## 7. 包的组织与 `__init__.py`

```python
# myapp/__init__.py —— 定义包的公开 API
from myapp.core.db import get_session
from myapp.core.cache import get_cache

__all__ = ["get_session", "get_cache"]
__version__ = "1.0.0"
```

好处：调用方写 `from myapp import get_session`，内部结构可以自由重构。
代价：**`import myapp` 会连带执行所有这些子模块的导入**——大包会拖慢启动。

按需惰性导入（3.7+ 的 `__getattr__`，PEP 562）：

```python
# myapp/__init__.py
def __getattr__(name):
    if name == "heavy":
        from myapp import heavy as m
        return m
    raise AttributeError(name)
```

## 8. 实用内省

```python
mod.__name__       # 模块全名
mod.__file__       # 文件路径（内建模块没有）
mod.__package__
mod.__doc__
dir(mod)
vars(mod)          # == mod.__dict__

import importlib.util
importlib.util.find_spec("numpy")    # 检查包是否可导入而不真的导入（None = 没装）
```

条件依赖的标准写法：

```python
try:
    import orjson as json_lib
except ImportError:
    import json as json_lib
```

## 9. 项目结构推荐（src layout）

```text
myproject/
├── pyproject.toml
├── src/
│   └── myapp/               # ← 包放在 src 下
│       ├── __init__.py
│       └── main.py
└── tests/
    └── test_main.py
```

**为什么用 src layout**：防止"测试时导入的是源码目录而非已安装的包"，
从而暴露打包配置错误。这是现代 Python 项目的主流做法，见 [[engineering/packaging-envs]]。

## 相关

- [[engineering/packaging-envs]] —— pyproject.toml、可编辑安装、虚拟环境
- [[language/scope-closure]] —— 模块级作用域
- [[concurrency/multiprocessing]] —— spawn 模式为什么必须有 `__main__` 守卫
- [[web/fastapi-architecture]] —— 大型 FastAPI 项目的模块划分
- [[bridge/npm-vs-pip]] —— 与 Node 模块解析的对照
