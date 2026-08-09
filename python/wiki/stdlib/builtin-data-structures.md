---
title: "内置数据结构：实现与复杂度"
date: 2026-08-07
tags: [list, dict, set, tuple, 哈希表, 复杂度, 数据结构]
sources: ["interview-python-cn.md", "python-cheatsheet.md"]
---

# 内置数据结构：实现与复杂度

面试必考：**「dict 是怎么实现的？」「list 和 tuple 有什么区别？」「什么样的对象能当 dict 的键？」**

## 1. 复杂度速查表（背下来）

### list（动态数组）

| 操作 | 复杂度 | 说明 |
|---|---|---|
| `l[i]` 索引 | **O(1)** | 指针数组 |
| `l[i] = x` | O(1) | |
| `l.append(x)` | **摊还 O(1)** | 过度分配，见 [[internals/memory-model]] |
| `l.pop()` | O(1) | 末尾 |
| `l.pop(0)` / `l.insert(0,x)` | **O(n)** ⚠️ | 要移动所有元素 → 用 `deque` |
| `x in l` | **O(n)** ⚠️ | 线性扫描 → 用 `set` |
| `l.remove(x)` / `del l[i]` | O(n) | |
| `len(l)` | O(1) | 长度是存下来的 |
| `l[a:b]` 切片 | O(k) | 复制 k 个指针 |
| `l.sort()` | O(n log n) | Timsort，稳定 |
| `min/max/sum` | O(n) | |

### dict / set（哈希表）

| 操作 | 平均 | 最坏 |
|---|---|---|
| `d[k]` / `k in d` / `d[k]=v` / `del d[k]` | **O(1)** | O(n)（全部哈希冲突） |
| 遍历 | O(n) | |
| `len(d)` | O(1) | |

### tuple / str

不可变 → 索引 O(1)，`in` 对 tuple 是 O(n)、对 str 是 O(n·m)（子串搜索）。

### deque（双端队列，双向链表块）

| 操作 | 复杂度 |
|---|---|
| `append` / `appendleft` / `pop` / `popleft` | **O(1)** |
| `d[i]` 随机索引 | **O(n)** ⚠️（与 list 相反） |

> **面试落点**：这张表最常被追问的两条：
> **① `list.pop(0)` 是 O(n)，队列要用 `collections.deque`。**
> **② `x in list` 是 O(n)、`x in set/dict` 是 O(1)** ——把 `in list` 换成 `in set`
> 是最常见的算法题优化点，也是实际代码 review 的高频意见。

## 2. list 的底层

```c
typedef struct {
    PyObject_VAR_HEAD
    PyObject **ob_item;    // 指向【指针数组】
    Py_ssize_t allocated;  // 已分配容量（≥ ob_size）
} PyListObject;
```

要点：

- 存的是**指针**，不是值 → 可以混装任意类型，但内存不连续、缓存不友好。
- `allocated ≥ len` 的差额就是过度分配的余量。
- 因此 **list ≠ C 数组 ≠ NumPy 数组**。数值计算必须用 `array`/`numpy`。

```python
# 混装是合法的（但类型注解和可读性上应避免）
[1, "a", None, [2]]
```

## 3. tuple vs list（经典题）

| | list | tuple |
|---|---|---|
| 可变 | ✅ | ❌ |
| 可哈希 | ❌ | ✅（元素全可哈希时） |
| 能做 dict 键 / set 元素 | ❌ | ✅ |
| 内存 | 更大（有 allocated 余量） | 更紧凑 |
| 创建速度 | 慢 | 快（有 freelist、常量可在编译期折叠） |
| 语义 | **同质集合，数量可变** | **异质记录，结构固定** |

```python
import timeit
timeit.timeit("[1,2,3]")      #=> ~0.02 µs
timeit.timeit("(1,2,3)")      #=> ~0.005 µs   常量 tuple 直接从常量池取
```

> **面试落点**：不要只答"一个可变一个不可变"。要答**语义差异**：
> list 表达"同类元素的序列"，tuple 表达"固定结构的记录"（像轻量 struct）。
> 再补一句「tuple 可哈希所以能做 dict 键，但前提是元素全可哈希」。

## 4. dict 的实现（重点）

### 开放寻址 + 紧凑布局

CPython 的 dict 用**开放寻址（open addressing）**，不是链地址法。
自 3.6 起用 **紧凑字典（compact dict）** 布局：

```text
indices:  [ -1, 1, -1, -1, 0, -1, 2, -1 ]        ← 稀疏的索引数组（int8/16/32，很小）
entries:  [ (hash0, key0, val0),                  ← 紧凑的条目数组，按插入顺序
            (hash1, key1, val1),
            (hash2, key2, val2) ]
```

带来三个结果：

1. **内存减少 20~25%**（稀疏数组存的是小整数索引，不是三元组）。
2. **保持插入顺序**——3.6 是实现细节，**3.7 起写进语言规范**。
3. 遍历更快（顺序扫描紧凑数组）。

> **面试落点**：「dict 是有序的吗？」——**Python 3.7+ 保证按插入顺序**，
> 这是紧凑字典布局的副产品，3.6 时还只是 CPython 实现细节。
> 3.7 之前需要顺序要用 `collections.OrderedDict`。
> （`OrderedDict` 至今仍有价值：它有 `move_to_end()`、`popitem(last=False)`，
> 且 `==` 比较**考虑顺序**。）

### 查找流程

```text
1. h = hash(key)
2. i = h & (table_size - 1)          ← 取低位做索引（容量是 2 的幂）
3. 若槽位空 → 未找到
4. 若槽位有条目：先比 hash 是否相等（快），再比 key 是否 `is` 同一对象（更快），
   最后才调 __eq__（慢）
5. 冲突则探测下一个位置（CPython 用扰动序列 perturb，兼顾随机性与局部性）
```

关键优化：**先比 hash 再比 `is` 最后比 `==`**——这就是字符串驻留能加速 dict 的原因。

### 扩容

负载因子超过 **2/3** 就扩容（新容量约为 `used * 3`，且是 2 的幂）。
扩容需要**重新哈希所有条目**——所以大 dict 的批量插入建议预知规模或直接用
`dict(zip(...))` / 推导式一次构建。

### 键的要求：可哈希（hashable）

```python
hash(obj)      # 必须可调用
# 可哈希 ⟺ 有 __hash__ 且不为 None，且需要 __eq__ 保持一致
```

**规则**：

- 不可变内置类型都可哈希：`int` `float` `str` `bytes` `tuple`(元素也可哈希) `frozenset` `None` `bool`
- 可变内置类型都**不可**哈希：`list` `dict` `set` `bytearray`
- 自定义类**默认可哈希**（按 `id()`），但**定义了 `__eq__` 就会失去 `__hash__`**

```python
{[1]: "x"}              # ❌ TypeError: unhashable type: 'list'
{(1, [2]): "x"}         # ❌ tuple 里有 list，仍不可哈希
{frozenset([1,2]): "x"} # ✅
```

一致性契约（必须遵守）：

```text
a == b  ⟹  hash(a) == hash(b)          （反之不必）
对象在字典中期间，其 hash 不能改变      （所以键应该是不可变的）
```

违反的后果：

```python
class Bad:
    def __init__(self, v): self.v = v
    def __hash__(self): return hash(self.v)
    def __eq__(self, o): return self.v == o.v

b = Bad(1)
d = {b: "x"}
b.v = 2                 # ❌ 改了参与 hash 的字段
d[b]                    # KeyError！键"丢失"在旧桶里了
```

### 哈希随机化（安全特性）

```console
$ python3 -c "print(hash('a'))"
-6785036359035667445
$ python3 -c "print(hash('a'))"
5124321927243512034      ← 每次进程启动都不同
```

`str`/`bytes` 的哈希加了随机盐（PYTHONHASHSEED），防止**哈希碰撞 DoS 攻击**
（构造大量同槽位的键让 dict 退化成 O(n)）。

- **同一进程内一致**，跨进程不一致。
- 因此 **`set` 的遍历顺序在不同运行间可能不同** → 不要依赖；测试需要确定性时设
  `PYTHONHASHSEED=0`。
- `int` 的 hash 没有随机化（`hash(1) == 1`，`hash(-1) == -2` 是个著名特例）。

## 5. set / frozenset

同样是哈希表，只是没有 value。

```python
a, b = {1,2,3}, {2,3,4}
a | b        # 并 union
a & b        # 交 intersection
a - b        # 差 difference
a ^ b        # 对称差
a <= b       # 子集 issubset
a.isdisjoint(b)

a.add(x); a.discard(x)   # discard 不存在不报错；remove 会 KeyError
frozenset([1,2])         # 不可变版，可哈希，能做 dict 键
```

**去重的正确姿势**：

```python
list(set(items))          # ❌ 顺序不确定
list(dict.fromkeys(items))# ✅ 保序去重（利用 dict 3.7+ 有序）
```

## 6. 该用哪个：决策表

| 需求 | 选择 |
|---|---|
| 有序序列、频繁末尾增删、随机索引 | `list` |
| 固定结构的记录、要当 dict 键 | `tuple` / `NamedTuple` |
| 键值映射、O(1) 查找 | `dict` |
| 成员判断、去重、集合运算 | `set` |
| 队列 / 双端 / 固定长度滑窗 | `collections.deque(maxlen=N)` |
| 优先队列 / TopK | `heapq` |
| 计数 | `collections.Counter` |
| 分组 | `collections.defaultdict(list)` |
| 有序插入位置 / 二分 | `bisect` |
| 大量同质数值 | `array.array` / `numpy.ndarray` |
| 需要有序键（按 key 排序） | 无内置！用 `sortedcontainers.SortedDict`（第三方） |

> **面试落点**：Python **没有内置的有序映射/平衡树**（对标 C++ 的 `std::map`、Java 的 `TreeMap`）。
> 需要时用第三方 `sortedcontainers`（纯 Python 但极快）或用 `heapq`/`bisect` 手工维护。
> 能主动指出这个"标准库空白"是懂行的表现。

## 7. 常见性能陷阱

```python
# ❌ O(n²)：list 里做成员判断
seen = []
for x in items:
    if x not in seen:      # O(n)
        seen.append(x)
# ✅ O(n)
seen = set()
for x in items:
    if x not in seen:
        seen.add(x)

# ❌ O(n²)：从头 pop
while lst: lst.pop(0)
# ✅
from collections import deque
q = deque(lst)
while q: q.popleft()

# ❌ O(n²)：循环里拼字符串 / 拼 list
s = ""
for x in xs: s += x
# ✅
"".join(xs)
list(itertools.chain.from_iterable(lists))

# ❌ 重复计算 len / 反复查全局
for i in range(len(xs)):    # 用 enumerate
    ...

# ❌ 在遍历时修改容器
for x in lst:
    if cond(x): lst.remove(x)    # RuntimeError 或跳过元素
# ✅
lst = [x for x in lst if not cond(x)]
for k in list(d):                # 需要改 dict 时先物化 keys
    if cond(k): del d[k]
```

## 相关

- [[stdlib/collections-itertools]] —— deque/Counter/defaultdict/heapq/bisect
- [[internals/memory-model]] —— list/dict 的扩容与内存
- [[internals/cpython-object-model]] —— 对象指针数组
- [[language/data-model]] —— `__hash__` / `__eq__` 契约
- [[interview/coding-patterns]] —— 算法题里的数据结构选型
- [[interview/question-bank-language]] —— dict 实现相关面试题
