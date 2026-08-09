---
title: "字符串、字节与编码"
date: 2026-08-07
tags: [字符串, bytes, 编码, unicode, f-string, 格式化]
sources: ["python-cheatsheet.md", "interview-python-cn.md"]
---

# 字符串、字节与编码

`str` / `bytes` 的严格分离是 Python 3 相对 Python 2 **最重要的破坏性变更**，
也是面试里"Python 2 和 3 的区别"的标准答案第一条。

## 1. `str` vs `bytes`

```python
s = "héllo"            # str：Unicode 码点序列（抽象文本）
b = b"hello"           # bytes：字节序列（不可变）
ba = bytearray(b"x")   # 可变的字节序列

s.encode("utf-8")      #=> b'h\xc3\xa9llo'    文本 → 字节
b"h\xc3\xa9".decode("utf-8")   #=> 'hé'       字节 → 文本

len("héllo")           #=> 5   码点数
len("héllo".encode())  #=> 6   字节数（é 占 2 字节）

s + b                  # ❌ TypeError: can only concatenate str (not "bytes") to str
```

**记忆法**：**encode = 编码成机器用的字节（往外走），decode = 解码成人看的文本（往里走）**。

> **面试落点**：Python 3 里 `str` 是 Unicode、`bytes` 是字节，**二者不能隐式转换**。
> Python 2 的 `str` 其实是字节串、`unicode` 才是文本，隐式转换导致了臭名昭著的
> `UnicodeDecodeError: 'ascii' codec can't decode byte`。Python 3 的严格分离在编译期就
> 消灭了这类 bug。

### I/O 边界上的规则

```python
open(p)                        # 文本模式：读出 str，用 locale 默认编码（危险！）
open(p, encoding="utf-8")      # ✅ 永远显式指定编码
open(p, "rb")                  # 二进制模式：读出 bytes
open(p, encoding="utf-8", errors="replace")   # 遇到非法字节替换成 �

# 网络/文件/数据库交互的一律是 bytes；处理逻辑一律用 str
# → 在【最外层边界】完成 decode/encode，中间层只处理 str
```

`errors` 参数取值：`strict`（默认，抛异常）、`ignore`、`replace`、`surrogateescape`（可逆保留非法字节）、
`backslashreplace`。

### 编码常识

| 编码 | 说明 |
|---|---|
| **UTF-8** | 变长 1~4 字节，ASCII 兼容，**唯一正确的默认选择** |
| UTF-16 | 变长 2/4 字节，有 BOM 和字节序问题 |
| GBK / GB18030 | 中文遗留编码，处理老系统数据时会遇到 |
| Latin-1 | 单字节，能"解码任何字节而不报错"（调试时的万能钥匙，但会得到乱码） |

```python
import sys, locale
sys.getdefaultencoding()          #=> 'utf-8'（str.encode 的默认）
locale.getpreferredencoding()     # open() 文本模式的默认 —— 平台相关！Windows 上可能是 cp936
# Python 3.15 起 open() 默认改为 UTF-8（PEP 686）；此前用 PYTHONUTF8=1 或显式传 encoding
```

## 2. 格式化：f-string 为王

```python
name, n, pi = "bob", 42, 3.14159

f"hi {name}, {n} items"           # ✅ 首选，最快
f"{n:>10}"                        # 右对齐宽 10
f"{n:08.2f}"                      # 补零，2 位小数 → '00042.00'
f"{pi:.3f}"                       #=> '3.142'
f"{1234567:,}"                    #=> '1,234,567' 千分位（注意 n=42 时只会得到 '42'）
f"{0.256:.1%}"                    #=> '25.6%'
f"{n:#x} {n:#b} {n:o}"            #=> '0x2a 0b101010 52'
f"{name!r}"                       # 用 repr()
f"{name=}"                        #=> "name='bob'"   3.8+ 调试神器
f"{value:{width}.{prec}f}"        # 嵌套格式说明符
f"{d['key']}"                     # 3.12+ 允许同种引号嵌套

# 旧写法（读老代码要认识）
"hi {}, {} items".format(name, n)
"hi %s, %d items" % (name, n)
Template("hi $name").substitute(name=name)   # string.Template，适合用户提供的模板
```

> ⚠️ **日志不要用 f-string**：`log.info(f"user {uid}")` 会**无条件**格式化字符串，
> 即使日志级别过滤掉了这条。用 `log.info("user %s", uid)` 让 logging 惰性格式化。
> 这是代码评审高频意见，也是面试细节题。
>
> 前端类比：f-string ≈ JS 模板字符串 `` `hi ${name}` ``，但**格式化能力强得多**
> （对齐、进制、千分位、精度全内建，不需要 Intl 或第三方库）。

## 3. 常用字符串方法

```python
s.strip() / .lstrip() / .rstrip()       # 去空白（可传要去掉的字符集）
s.removeprefix("http://")               # 3.9+，比 s[7:] 安全
s.removesuffix(".txt")
s.split(",") / .rsplit(",", 1) / .splitlines()
s.partition("=")                        #=> ('k', '=', 'v')  只切一次，永远返回 3 元组
",".join(parts)
s.replace(a, b, count)
s.startswith(("http://", "https://"))   # 可传元组
s.find(x)      # -1 表示没找到
s.index(x)     # 抛 ValueError
s.count(x)
s.upper() / .lower() / .casefold()      # casefold 更彻底（用于大小写无关比较）
s.title() / .capitalize()
s.center(20, "-") / .ljust(10) / .zfill(5)
s.isdigit() / .isalpha() / .isalnum() / .isspace() / .isidentifier()
s.encode() / b.decode()
s.translate(str.maketrans("abc", "xyz"))
s.format_map(d)
```

## 4. 不可变性与性能

`str` 不可变，所以**循环里 `+=` 是 O(n²)**：

```python
# ❌ 每次都创建新字符串
s = ""
for x in items:
    s += str(x)

# ✅ O(n)
s = "".join(str(x) for x in items)

# ✅ 增量构建用 list 或 io.StringIO
buf = []
for x in items: buf.append(str(x))
s = "".join(buf)
```

> CPython 对 `s += x` 在**引用计数为 1** 时有原地扩容优化，所以你可能测不出 O(n²)——
> 但这是脆弱的实现细节（PyPy 上就没有），且一旦有第二个引用就退化。**永远用 join。**

## 5. 切片（Python 独有的强大之处）

```python
s = "abcdefg"
s[2]        #=> 'c'
s[-1]       #=> 'g'
s[1:4]      #=> 'bcd'      [start:stop)
s[:3]       #=> 'abc'
s[3:]       #=> 'defg'
s[::2]      #=> 'aceg'     步长
s[::-1]     #=> 'gfedcba'  反转
s[10:20]    #=> ''         越界切片不报错！（但 s[10] 报 IndexError）

# 切片对象可以复用
head = slice(0, 3)
s[head]     #=> 'abc'

# 列表切片赋值（str 不行，list 可以）
xs = [1,2,3,4,5]
xs[1:3] = [9]        #=> [1, 9, 4, 5]      长度可变
xs[::2] = [0, 0, 0]  # 扩展切片赋值长度必须匹配
del xs[1:3]
```

> **面试落点**：`s[10:20]` 不报错而 `s[10]` 报错——切片会自动裁剪到边界。
> 这个不对称是常考的细节。

## 6. 正则要点

```python
import re

m = re.search(r"(\d{4})-(\d{2})", text)     # 找第一个
if m:
    m.group(0), m.group(1), m.groups(), m.span()

re.match(r"\d+", s)          # 只从【开头】匹配（不是全串！全串用 fullmatch）
re.fullmatch(r"\d+", s)
re.findall(r"\d+", s)        # 返回字符串列表（有分组时返回分组元组）
re.finditer(r"\d+", s)       # 惰性，返回 Match 对象
re.sub(r"\s+", " ", s)
re.sub(r"(\w+)@(\w+)", r"\2:\1", s)         # 反向引用
re.split(r"[,;]", s)

pat = re.compile(r"^(?P<key>\w+)=(?P<val>.*)$", re.M)    # 预编译 + 命名分组
for m in pat.finditer(text):
    m["key"], m["val"]

# 常用 flag
re.I  # IGNORECASE
re.M  # MULTILINE：^ $ 匹配每行
re.S  # DOTALL：. 匹配换行
re.X  # VERBOSE：允许写注释和空白
```

**永远用原始字符串 `r"..."` 写正则**，否则 `\d` 之类会被 Python 先转义一遍。

## 7. Python 2 vs 3 的字符串差异（面试常问）

| | Python 2 | Python 3 |
|---|---|---|
| `str` | 字节串 | Unicode 文本 |
| `unicode` | 文本类型 | 已移除（合并进 str） |
| `bytes` | `str` 的别名 | 独立类型 |
| 字面量 | `"x"` 是字节，`u"x"` 是文本 | `"x"` 是文本，`b"x"` 是字节 |
| 隐式转换 | 会（用 ASCII，常炸） | **不会**，必须显式 encode/decode |
| `print` | 语句 | 函数 |
| 默认源码编码 | ASCII（要写 `# -*- coding: utf-8 -*-`） | UTF-8 |

Python 2 已于 2020-01-01 EOL，新项目绝不使用；但面试仍会问差异，
用来判断你是否理解 Unicode 模型。

## 相关

- [[language/objects-mutability]] —— 字符串驻留（interning）
- [[language/comprehensions-functional]] —— join / 字符串处理惯用法
- [[internals/cpython-object-model]] —— CPython 的紧凑 Unicode 表示（PEP 393）
- [[stdlib/stdlib-essentials]] —— pathlib / json / re 的完整用法
- [[interview/question-bank-language]] —— 编码相关面试题
