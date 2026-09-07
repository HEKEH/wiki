---
title: "复习题组 05 —— collections / itertools / functools / heapq / bisect 自测卷（含批改记录）"
date: 2026-09-06
tags: [面试题, 复习, collections, itertools, functools, heapq, bisect, 标准库]
sources: []
---

# 复习题组 05 —— 标准库工具箱自测卷

主题页是 [[stdlib/collections-itertools]]，本卷 **50 题**（选择 20 / 判断 15 / 填空 15）
基本覆盖该页全部考点：`collections` 六件套、`itertools` 惰性迭代器代数、
`functools` 八件套、`heapq` 与 `bisect`。

**用法**：闭卷作答，把答案直接写在每题下方的 `**答：**` 后面（`_` 处填内容）。
判断题写 `✅` / `❌`。全部写完后交给我批改——我会逐题给对错、补解析和面试落点，
并把批改记录追加到本页末尾。

**不确定的题也要写**，并在答案后加 `(?)`，这样能区分「真会」和「蒙对」。

| 分区 | 题号 | 覆盖 |
|---|---|---|
| 选择题 | 1–20 | 六件套语义、itertools 输出推演、functools 陷阱、heapq/bisect API |
| 判断题 | 1–15 | 最容易记反的机制性结论 |
| 填空题 | 1–15 | 术语、复杂度、版本号、惯用写法 |

---

## 一、选择题（20 题，每题一个正确答案）

**1.** 对 `d = defaultdict(list)`，以下哪个操作**会**真的把 `"k"` 插入字典？

A. `d.get("k")`　B. `"k" in d`　C. `d["k"]`　D. 以上都不会

**答：** C

**2.** `a = Counter("aab")`，`b = Counter("abc")`，则 `a - b` 是：

A. `Counter({'a': 1, 'b': 0, 'c': -1})`　B. `Counter({'a': 1})`
C. `Counter({'a': 1, 'b': 0})`　D. 抛 `ValueError`

**答：** A

**3.** `Counter.most_common(n)`（传了 `n`）内部使用的算法与复杂度是：

A. `sorted`，O(n log n)　B. `heapq.nlargest`，O(n log k)
C. 计数排序，O(n)　D. 插入排序，O(n²)

**答：** B

**4.** `d = deque([1,2,3], maxlen=3)`，执行 `d.append(4)` 后 `d` 是：

A. `deque([1,2,3])`　B. `deque([2,3,4])`　C. `deque([1,2,3,4])`　D. 抛 `IndexError`

**答：** B

**5.** `d = deque([1,2,3])`，执行 `d.extendleft([4,5])` 后 `d` 是：

A. `deque([4,5,1,2,3])`　B. `deque([5,4,1,2,3])`
C. `deque([1,2,3,4,5])`　D. `deque([1,2,3,5,4])`

**答：** B

**6.** 关于 `deque` 的复杂度，正确的是：

A. 两端增删 O(1)，随机索引 O(n)　B. 两端增删 O(n)，随机索引 O(1)
C. 两者都是 O(1)　D. 两者都是 O(n)

**答：** A

**7.** `Point = namedtuple("Point", "x y")`，则 `Point(1, 2) == (1, 2)` 的结果是：

A. `True`　B. `False`　C. `TypeError`　D. 取决于 Python 版本

**答：** A

**8.** `cfg = ChainMap(cli, env, defaults)`，执行 `cfg["port"] = 1234` 后被修改的是：

A. `defaults`　B. `env`　C. `cli`　D. 三个都被修改

**答：** C

**9.** Python 3.7+ 普通 dict 已保证插入序，下列**不属于** `OrderedDict` 独有能力的是：

A. `move_to_end`　B. `popitem(last=False)`　C. 顺序敏感的 `==`　D. 键的哈希去重

**答：** D

**10.** `list(it.islice("ABCDEFG", 1, 6, 2))` 的结果是：

A. `['A','C','E']`　B. `['B','D','F']`　C. `['B','C','D','E','F']`　D. `['A','B','C']`

**答：** B

**11.** `list(it.chain.from_iterable([[1, [2]], [3]]))` 的结果是：

A. `[1, 2, 3]`　B. `[1, [2], 3]`　C. `[[1, [2]], [3]]`　D. `[1, 2, [3]]`

**答：** B

**12.** `list(it.dropwhile(lambda x: x < 3, [1,2,3,1,2]))` 的结果是：

A. `[1,2,1,2]`　B. `[3]`　C. `[3,1,2]`　D. `[1,2]`

**答：** C

**13.** `list(it.accumulate([3,1,4,1,5], max))` 的结果是：

A. `[3,4,8,9,14]`　B. `[3,3,4,4,5]`　C. `[5,5,5,5,5]`　D. `[3,1,4,1,5]`

**答：** B

**14.** 关于 `itertools.groupby`，正确的说法是：

A. 它会自动按 key 排序后分组　B. 它只把**相邻的**、key 相同的元素归为一组
C. 它返回的 group 是 list，可反复遍历　D. 它与 SQL 的 `GROUP BY` 语义一致

**答：** B

**15.** `@cache` 等价于下列哪个写法？

A. `@lru_cache(maxsize=128)`　B. `@lru_cache(maxsize=None)`
C. `@cached_property`　D. `@lru_cache(typed=True)`

**答：** A

**16.** 用 `@cache` 装饰**实例方法**的主要问题是：

A. 参数不可哈希　B. `self` 被存进函数级全局缓存，实例永不被回收（内存泄漏）
C. 与 `__slots__` 冲突　D. 无法调用 `cache_clear()`

**答：** B

**17.** `functools.partial` 相比 `lambda` 的两大优势是：

A. 更快、可 pickle　B. 更快、支持类型注解
C. 可 pickle、支持异步　D. 更省内存、线程安全

**答：** A

**18.** `@singledispatchmethod` 分派时依据的是：

A. `self` 的类型　B. 第一个参数（即 `self`）的类型
C. 跳过 `self`，按第二个参数的类型　D. 返回值类型

**答：** C

**19.** `heapq.heapreplace` 与 `heapq.heappushpop` 的关键区别是：

A. 前者先 pop 后 push，堆为空会报错；后者先 push 后 pop，堆为空不报错
B. 前者只能用于最大堆　C. 后者不保证堆序　D. 两者完全等价

**答：** A

**20.** `xs = [1,3,5,5,7]`，`bisect.bisect_right(xs, 5)` 的返回值是：

A. `2`　B. `3`　C. `4`　D. `5`

**答：** D

---

## 二、判断题（15 题，写 ✅ 或 ❌；判错的请补一句正确说法）

**1.** `defaultdict` 的机制是「取值时返回一个默认值」，并不会真的往字典里写入条目。

**答：** ❌

**2.** `Counter.update()` 是累加计数，而不是像 `dict.update()` 那样覆盖。

**答：** ✅

**3.** `a.subtract(b)` 会丢弃结果为负的计数，并返回一个新的 `Counter`。

**答：** ❌

**4.** `d.rotate(1)` 是把 deque 向**左**旋转一位。

**答：** ❌

**5.** `ChainMap` 会把传入的多个 dict 拷贝合并成一份，此后修改原 dict 不影响它。

**答：** ❌

**6.** `itertools` 的函数返回的都是迭代器，不 `list()` 就不会真正计算，且只能消费一次。

**答：** ✅

**7.** `it.chain(*lists)` 与 `it.chain.from_iterable(lists)` 结果相同，但后者是惰性的，能接受无限长的 `lists`。

**答：** ✅

**8.** 调用 `it.tee(src)` 之后，仍可安全地继续从原迭代器 `src` 取元素。

**答：** ❌

**9.** `islice` 支持负索引，并且能 O(1) 跳过前面的元素。

**答：** ❌

**10.** `groupby` 产出的 group 是共享底层迭代器的惰性对象，跨组保存后再遍历会得到空结果。

**答：** ✅

**11.** `cached_property` 把结果写进实例 `__dict__`，因此与 `__slots__` 不兼容，且不保证线程安全。

**答：** ✅

**12.** 写装饰器不加 `@wraps`，会让被装饰函数的 `__name__` / `__doc__` / 签名信息错乱，影响 `help()`、pytest、FastAPI 等。

**答：** ✅

**13.** `heapq.heapify(h)` 之后，`h` 是一个完全排好序的 list。

**答：** ❌

**14.** 对已排序的 list 使用 `in` 运算符，复杂度仍然是 O(n)。

**答：** ✅

**15.** `bisect.insort` 的整体复杂度是 O(log n)。

**答：**❌

---

## 三、填空题（15 题）

**1.** `defaultdict` 自动创建默认值，靠的是 dict 的 `______________` 钩子方法。

**答：**

**2.** 一元运算 `+Counter(a=1, b=-1, c=0)` 的结果是 `______________`，它的典型用途是 ______________。

**答：** Counter(a=1)  去掉count为0或负数的项

**3.** 判断两个字符串是否为变位词（anagram），一行写法是 `__________("listen") == __________("silent")`。

**答：** Counter Counter

**4.** 被问「Python 怎么实现队列」应答 `______________`；不能答 `list.pop(0)`，因为它是 O(____) 的。

**答：** deque  n

**5.** 手写 LRU 缓存的两个关键操作是 `OrderedDict.______________` 和 `popitem(last=______)`。

**答：** move_to_end  False

**6.** 展平一层嵌套列表最地道的写法是 `itertools.______________`。

**答：** from_iterable

**7.** `list(it.product("AB", repeat=2))` 共产出 ____ 个元素；一般地 `product(xs, repeat=r)` 的规模是 ____________。

**答：** 4  len(xs)的r次方

**8.** 相邻元素两两配对用 `itertools.__________`（____+ 引入）；定长分批用 `itertools.__________`（____+ 引入）。

**答：** pairwise

**9.** 数据无序却想按 key 分组时，比 `groupby` 更直接也更快的做法是用 `______________`，复杂度 O(____) vs 需先排序的 O(__________)。

**答：** _

**10.** 元素可哈希且不需要 key 函数时，「去重保序」最短的写法是 `list(______________(xs))`。

**答：** dict.fromkeys

**11.** `lru_cache` 的底层实现是 ____________ + ____________，命中复杂度 O(____)。

**答：** OrderedDict

**12.** 加了 `@wraps` 之后，可以通过 `______________` 属性拿回未被装饰的原函数。

**答：** _

**13.** Python 只有最小堆，实现最大堆的技巧是 ______________；元素是 tuple 时写成 `(____________, item)`。

**答：** 加负号    index

**14.** 堆中元素不可比较时，用 `itertools.__________` 生成单调序号打破平局，同时顺带实现了同优先级下的 ________ 顺序。

**答：** _

**15.** 有序序列中值 `v` 出现的次数 = `bisect.______________(xs, v)` − `bisect.______________(xs, v)`。

**答：** bisect_right bisect_left

---

## 参考答案与批改

**批改日期**：2026-09-07　**得分**：选择 17/20 · 判断 15/15 · 填空 8.25/15（合计 40.25/50）

判断题满分，选择题只错 3 道且都是「记忆型」失分；填空题的失分**集中在四个空题**——
不是记错，是**没记住**。下面只对填空题逐题批改（选择题错处附在末尾）。

### 三、填空题逐题批改

| # | 你的答案 | 判定 | 正确答案 |
|---|---|---|---|
| 1 | （空） | ❌ | `__missing__` |
| 2 | `Counter(a=1)`；去掉 count 为 0 或负数的项 | ✅ | 同 |
| 3 | `Counter` / `Counter` | ✅ | 同 |
| 4 | `deque`；O(n) | ✅ | 同 |
| 5 | `move_to_end`；`False` | ✅ | 同 |
| 6 | `from_iterable` | ⚠️ 半对 | `chain.from_iterable` |
| 7 | 4；`len(xs)` 的 r 次方 | ✅ | 4；n^r |
| 8 | `pairwise` | ⚠️ 1/4 空 | `pairwise`（**3.10**+）；`batched`（**3.12**+） |
| 9 | （空） | ❌ | `defaultdict(list)`；O(**n**) vs O(**n log n**) |
| 10 | `dict.fromkeys` | ✅ | 同 |
| 11 | `OrderedDict` | ❌ | **dict** + **双向循环链表**；O(**1**) |
| 12 | （空） | ❌ | `__wrapped__` |
| 13 | 加负号；`index` | ⚠️ 半对 | 取负数；`(-priority, item)` |
| 14 | （空） | ❌ | `count`；**FIFO** |
| 15 | `bisect_right` − `bisect_left` | ✅ | 同（3.11.9 实测 `= 2`） |

### 需要重点回炉的五处

**① 第 1 题 · `__missing__`（空题）**

这是 `defaultdict` 唯一的机制，也是选择题第 1 题的原理。`dict.__getitem__` 找不到键时
会回调 `__missing__`，`defaultdict` 在里面做了两件事：调 `default_factory()` 造值、
**写回字典**、再返回。所以「读一下」会产生副作用：

```python
d = defaultdict(list)
bool(d["missing"])   #=> False   条件为假
"missing" in d       #=> True    但键已经被插进去了
```

`__missing__` 是 dict 的钩子，不是 defaultdict 私有的——继承 `dict` 自己实现 `__missing__`
也能做出同样效果，这是面试里的加分追问。

**② 第 6 题 · `chain.from_iterable`（半对）**

`itertools.from_iterable` 不存在。`from_iterable` 是挂在 `chain` 上的**类方法**，
完整路径 `itertools.chain.from_iterable`。记忆锚点：它和 `chain(*lists)` 结果相同，
差别在**惰性**——前者接受无限长的 `lists`，后者要先解包成参数（生成器会被立刻耗尽）。

**③ 第 8 题 · 版本号（只答出 1/4 空）**

版本号是这页最容易丢分的地方，四个一起背：

| API | 版本 | 一句话 |
|---|---|---|
| `itertools.accumulate(..., initial=)` | 3.8+ | 累积的初始值 |
| `Counter.total()` | 3.10+ | 所有计数之和 |
| `itertools.pairwise` | 3.10+ | 相邻两两配对（滑动窗口 n=2） |
| `bisect.*(..., key=)` | 3.10+ | 对对象列表按字段二分 |
| `functools.cache` | 3.9+ | = `lru_cache(maxsize=None)` |
| `itertools.batched` | 3.12+ | 定长分批，最后一批可能不足 |

**④ 第 11 题 · `lru_cache` 的底层（❌，但错得有价值）**

正确答案是 **dict + 双向循环链表**，命中 O(1)。你答 `OrderedDict` 说明方向是对的——
`OrderedDict` 本身**就是**「dict + 双向链表」，主题页也写了 `lru_cache`「和手写 LRU 同构」。
但要分清两件事：

- **手写 LRU**（面试手撕题）→ 用 `OrderedDict.move_to_end` + `popitem(last=False)`（第 5 题，你答对了）
- **`functools.lru_cache`** → C 实现，自己维护一条循环双向链表 + 一个 dict，**不用 `OrderedDict`**

面试落点也在这里：它是**挂在函数对象上的全局字典**，所以装饰实例方法会让 `self` 常驻 →
内存泄漏（选择题第 16 题，你答对了）。这三题其实是同一条链，把第 11 题补上就串起来了。

**⑤ 第 13 题第二空 · `(-priority, item)`（半对）**

第一空「加负号」正确。第二空你写了 `index`，应该是把它和**第 14 题的序号技巧**混了——
这两个是不同问题，正好一起理清：

```python
# 问题 A：只有最小堆，要最大堆 → 优先级取负
heapq.heappush(h, (-priority, item))

# 问题 B：优先级相同时，堆会去比较第二项，item 可能不支持 < → TypeError
counter = itertools.count()
heapq.heappush(h, (priority, next(counter), task))
# 序号单调递增：既保证平局时可比较，又实现了同优先级下的 FIFO
```

两者可以叠加：`(-priority, next(counter), task)` 就是「最大堆 + 平局 FIFO」的完整写法。

**（第 12、14 题空题）**：`__wrapped__` 是 `@wraps` 顺手挂上的属性，`inspect.signature`
靠它还原原始签名（FastAPI 解析参数就依赖这条链）；第 14 题的 `itertools.count` 见上面 ④。

### 选择题的 3 处错误

| # | 你的答案 | 正确 | 为什么 |
|---|---|---|---|
| 2 | A | **B** `Counter({'a': 1})` | `Counter` 的**减法丢弃计数 ≤ 0 的项**，`b`、`c` 的结果分别是 0 和 -1，都被丢掉。保留负数要用 `a.subtract(b)`（原地修改）——判断题第 3 题你判对了，但没迁移到这道输出题上 |
| 15 | A | **B** `@lru_cache(maxsize=None)` | `@cache`（3.9+）是**无上限、不做淘汰**的版本，所以更快；`maxsize=128` 是 `lru_cache` 不传参时的默认值，不是 `cache` |
| 20 | D | **C** `4` | `xs = [1,3,5,5,7]`，两个 5 在下标 2、3；`bisect_right` 插到相等元素的**右**边 → 4（`5` 是列表长度，不是插入点）。配合填空第 15 题：出现次数 = 4 − 2 = 2 |

### 小结：回主题页重读哪几段

按性价比排序：

1. [[stdlib/collections-itertools]] **§3 functools** —— `lru_cache` 的实现与三个坑（第 11 题）、`wraps` 的 `__wrapped__`（第 12 题）
2. 同页 **§2 itertools** 的版本号标注 —— `pairwise` / `batched` / `accumulate(initial=)`（第 8 题）
3. 同页 **§4 heapq** 的「最大堆技巧」与「打破平局」两段 —— 分清两个不同问题（第 13、14 题）
4. 同页 **§1 collections** 的 `defaultdict`「面试落点」段 —— `__missing__` 的写入副作用（第 1 题）、以及 `Counter` 减法丢负数（选择题第 2 题）
5. 同页 **§6 速查表** —— 第 9 题的 `defaultdict(list)` vs `groupby` 就在表的第二行

判断题 15/15、选择题里 `deque` / `groupby` / `tee` / `islice` / `partial` /
`singledispatchmethod` / `heapreplace` 全对，说明**机制性理解是扎实的**，
失分几乎都在「具体名字、具体版本号、具体复杂度」这类可以纯靠背补上的点。

---

## 相关

- [[stdlib/collections-itertools]] —— 本卷的主题页
- [[stdlib/builtin-data-structures]] —— 复杂度与底层实现
- [[interview/coding-patterns]] —— 算法题里的应用
- [[interview/traps]] —— 更偏语言层的陷阱题
