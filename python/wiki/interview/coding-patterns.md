---
title: "手撕代码：Python 惯用法与模板"
date: 2026-08-07
tags: [算法, 手撕代码, LeetCode, 惯用法, 模板]
sources: ["python-cheatsheet.md"]
---

# 手撕代码：Python 惯用法与模板

面试白板/共享编辑器里，**代码是否 Pythonic 直接影响评价**。
同样 AC 的两份代码，用 `enumerate`/`Counter`/`deque` 的那份明显更专业。

## 1. 必须改掉的 JS 习惯

```python
# ❌ C/JS 式                          # ✅ Pythonic
for i in range(len(arr)):             for i, x in enumerate(arr):
    x = arr[i]

for i in range(len(a)):               for x, y in zip(a, b):
    a[i] + b[i]

if len(arr) == 0:                     if not arr:

i = 0                                 for x in arr:
while i < len(arr): ...                   ...

result = []                           result = [f(x) for x in arr if p(x)]
for x in arr:
    if p(x): result.append(f(x))

tmp = a; a = b; b = tmp               a, b = b, a

if x >= 0 and x < n:                  if 0 <= x < n:

d[k] = d.get(k, 0) + 1                cnt = Counter(arr)
                                      # 或 d = defaultdict(int); d[k] += 1

s = ""                                s = "".join(parts)
for x in xs: s += x

max_v = -float("inf")                 max_v = max(arr)
for x in arr:
    if x > max_v: max_v = x
```

## 2. 数据结构速用

```python
from collections import Counter, defaultdict, deque, OrderedDict
import heapq, bisect, itertools, math
from functools import lru_cache, cache, reduce

Counter(s).most_common(k)          # 频次 TopK
defaultdict(list)                  # 分组
deque()                            # BFS 队列 / 滑动窗口，两端 O(1)
heapq.heappush(h, (-v, x))         # 最大堆用负数
heapq.nlargest(k, arr)             # TopK
bisect.bisect_left(arr, x)         # 有序数组二分
math.inf / -math.inf
divmod(a, b)                       #=> (商, 余数)
```

## 3. 高频模板

### 双指针

```python
def two_sum_sorted(nums: list[int], target: int) -> tuple[int, int] | None:
    l, r = 0, len(nums) - 1
    while l < r:
        s = nums[l] + nums[r]
        if s == target: return l, r
        if s < target:  l += 1
        else:           r -= 1
    return None
```

### 滑动窗口

```python
def longest_unique(s: str) -> int:
    seen: dict[str, int] = {}
    left = best = 0
    for right, ch in enumerate(s):
        if ch in seen and seen[ch] >= left:
            left = seen[ch] + 1
        seen[ch] = right
        best = max(best, right - left + 1)
    return best
```

### 二分（记住这个不变式版本）

```python
def lower_bound(arr: list[int], target: int) -> int:
    """返回第一个 >= target 的下标"""
    lo, hi = 0, len(arr)              # 左闭右开
    while lo < hi:
        mid = (lo + hi) // 2          # Python 整数无溢出，不用 lo+(hi-lo)//2
        if arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo

# 标准库直接有
bisect.bisect_left(arr, target)       # == lower_bound
bisect.bisect_right(arr, target)      # == upper_bound
```

### BFS / DFS

```python
from collections import deque

def bfs(graph: dict, start):
    visited = {start}
    q = deque([start])
    while q:
        node = q.popleft()                  # ★ 不是 pop(0)
        for nxt in graph[node]:
            if nxt not in visited:
                visited.add(nxt)
                q.append(nxt)
    return visited

def bfs_levels(grid, start):
    q = deque([start]); seen = {start}; depth = 0
    while q:
        for _ in range(len(q)):             # 按层处理
            r, c = q.popleft()
            for dr, dc in ((0,1),(0,-1),(1,0),(-1,0)):
                nr, nc = r+dr, c+dc
                if 0 <= nr < len(grid) and 0 <= nc < len(grid[0]) and (nr,nc) not in seen:
                    seen.add((nr,nc)); q.append((nr,nc))
        depth += 1
    return depth

def dfs(node, visited=None):
    visited = visited if visited is not None else set()   # ★ 别用可变默认参数
    visited.add(node)
    for nxt in graph[node]:
        if nxt not in visited:
            dfs(nxt, visited)
    return visited

# 递归深度上限 1000，深图要改迭代版
import sys; sys.setrecursionlimit(10000)
```

### 动态规划 + 记忆化

```python
from functools import cache

@cache                              # ★ 一行搞定记忆化（参数须可哈希）
def fib(n: int) -> int:
    return n if n < 2 else fib(n-1) + fib(n-2)

# 滚动数组省空间
def climb(n: int) -> int:
    a, b = 1, 1
    for _ in range(n - 1):
        a, b = b, a + b
    return b

# 二维 DP
dp = [[0] * (n + 1) for _ in range(m + 1)]     # ★ 不能写 [[0]*n]*m！（行是同一个对象）
```

> **`[[0]*n]*m` 是必考陷阱**：`*m` 复制的是**同一个列表对象的引用**，
> 改 `dp[0][0]` 会让所有行都变。必须用列表推导式。

### 堆 / TopK

```python
# 第 K 大
heapq.nlargest(k, nums)[-1]
# 维护大小为 k 的最小堆（O(n log k)）
h = []
for x in nums:
    heapq.heappush(h, x)
    if len(h) > k: heapq.heappop(h)
h[0]                                # 第 k 大

# 合并 K 个有序链表/数组
list(heapq.merge(*lists))
```

### 前缀和 / 差分

```python
prefix = list(itertools.accumulate(nums, initial=0))
range_sum = prefix[r+1] - prefix[l]
```

### 回溯

```python
def permute(nums: list[int]) -> list[list[int]]:
    res, path, used = [], [], [False] * len(nums)
    def backtrack():
        if len(path) == len(nums):
            res.append(path[:])          # ★ 必须拷贝！否则存的是同一个引用
            return
        for i, x in enumerate(nums):
            if used[i]: continue
            used[i] = True; path.append(x)
            backtrack()
            path.pop(); used[i] = False  # 撤销
    backtrack()
    return res
```

### 链表与树

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val, self.next = val, next

def reverse_list(head):
    prev = None
    while head:
        head.next, prev, head = prev, head, head.next     # ★ 一行三重赋值
    return prev

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val, self.left, self.right = val, left, right

def inorder(root):
    return inorder(root.left) + [root.val] + inorder(root.right) if root else []
```

### LRU Cache（高频手写题）

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.d: OrderedDict[int, int] = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.d:
            return -1
        self.d.move_to_end(key)             # ★ OrderedDict 的杀手锏
        return self.d[key]

    def put(self, key: int, value: int) -> None:
        if key in self.d:
            self.d.move_to_end(key)
        self.d[key] = value
        if len(self.d) > self.cap:
            self.d.popitem(last=False)      # 弹出最久未用的
```

> 追问"不用 OrderedDict 怎么写？"→ 哈希表 + 双向链表，
> 这正是 `OrderedDict` 的内部实现。

## 4. 字符串处理

```python
s.lower() / .strip() / .split() / .replace()
s.isalnum() / .isdigit() / .isalpha()
"".join(reversed(s))  ==  s[::-1]
sorted(s)                              # 字母异位词判断：sorted(a) == sorted(b)
Counter(a) == Counter(b)               # 更快的异位词判断
ord("a") / chr(97)                     # 字符 ↔ 码点
[0] * 26                               # 小写字母计数数组
idx = ord(ch) - ord("a")
```

## 5. 输入输出（笔试题会用）

```python
import sys
data = sys.stdin.read().split()        # ★ 大量输入时比逐行 input() 快很多
n = int(data[0])

n, m = map(int, input().split())
arr = list(map(int, input().split()))
print(*arr)                            # 空格分隔输出
print("\n".join(map(str, arr)))
sys.setrecursionlimit(10**6)
```

## 6. 复杂度速记

| 结构 | 查找 | 插入 | 删除 | 备注 |
|---|---|---|---|---|
| list | O(n) | O(1) 尾 / O(n) 头 | O(n) | `pop(0)` 是 O(n) |
| deque | O(n) | **O(1) 两端** | O(1) 两端 | 随机索引 O(n) |
| dict/set | **O(1)** | O(1) | O(1) | 最坏 O(n) |
| heap | O(1) 查最小 | O(log n) | O(log n) | 建堆 O(n) |
| 有序 list + bisect | O(log n) | O(n) | O(n) | |

排序 `sorted`：**Timsort，O(n log n)，稳定**。

## 7. 面试现场的加分习惯

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    """哈希表一次遍历。时间 O(n)，空间 O(n)。"""
    seen: dict[int, int] = {}                    # ① 加类型注解
    for i, x in enumerate(nums):                 # ② 用 enumerate
        if (j := seen.get(target - x)) is not None:   # ③ 会用 walrus
            return [j, i]
        seen[x] = i
    return []                                     # ④ 处理无解情况
```

流程建议：

1. **先复述题目和边界**（空输入？重复元素？负数？超大输入？）
2. **先说思路和复杂度，再写**（"我打算用哈希表把 O(n²) 降到 O(n)"）
3. **写完主动跑几个例子**，包括边界
4. **说出可优化的方向**（"如果数组已排序可以用双指针把空间降到 O(1)"）

**不要**：一上来闷头写、忽略空输入、用 `pop(0)`、写 `[[0]*n]*m`、
在循环里 `s += x`、用可变默认参数。

## 相关

- [[stdlib/builtin-data-structures]] —— 复杂度与底层实现
- [[stdlib/collections-itertools]] —— 工具箱
- [[language/comprehensions-functional]] —— 惯用法
- [[interview/traps]] —— 陷阱题
- [[interview/roadmap]] —— 复习路线
