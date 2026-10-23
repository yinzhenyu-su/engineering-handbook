---
title: "Python、Java、C++、Go 的语言习惯写法"
description: "Python、Java、C++、Go 的习惯写法：集合、遍历、排序、比较器。每种语言希望你写出的方言。"
slug: 05-language-idioms
part: 0
chapter: 5
difficulty: beginner
prerequisites: [00-how-to-use]
date_updated: 2026-05-24
languages: [python, java, cpp, go]
ladder:
  core: []
  stretch: []
---

# Python、Java、C++、Go 的语言习惯写法

一位候选人在 Python 里写出显然的频率统计：`Counter(s)`，一行，提交。为了 Amazon 的面试轮换到 Java 移植，他写 `freq[c]++`。编译器拒绝。同样的形状放在 C++ 里直接就能跑；放在 Go 里也能跑。一个算法，四种表面形式，其中两种依赖另外两种没有的语言规则。本章的全部内容就在这个细节里。

这一页是你会反复回来查的参考。站点上后面每一章渲染的代码标签页都遵循这些约定；如果某个 Java 标签页用 `ArrayDeque` 实现队列，或者某个 Go 标签页在向下传递之前先克隆切片，依据的就是你现在读的这一页。下面八条习惯规则（DSH-01 到 DSH-08）由 CI 强制执行；本章给出的是理由和表格。

## 同一个问题，四种写法

统计 `"abracadabra"` 中每个字符的出现频率。答案是 `{a: 5, b: 2, r: 2, c: 1, d: 1}`。下面是最标准的 Python 写法，之所以短，是因为 `collections.Counter` 就是为这件事准备的。

```python
from collections import Counter


def count_freqs(s: str) -> dict[str, int]:
    return dict(Counter(s))
```

**解法** — Python（[Java](./05-language-idioms/sol.java) · [C++](./05-language-idioms/sol.cpp) · [Go](./05-language-idioms/sol.go)）

```python
from collections import Counter


def count_freqs(s: str) -> dict[str, int]:
    """Return a frequency map of characters in s using Counter."""
    return dict(Counter(s))
```

把四种写法并排看。形状在两个关键处出现分歧。Python 和 Java 把缺失键的检查写明白：`Counter` 把它藏起来，`getOrDefault(c, 0) + 1` 把它暴露出来。C++ 和 Go 让零值规则把这个检查整个消掉。`++freq[c]` 和 `freq[s[i]]++` 之所以成立，是因为在 `std::unordered_map` 中索引一个不存在的键会把新条目的值初始化为零[^1]，而在 `map[byte]int` 中则返回值类型的零值。[^2]

这一个观察可以推广到后面大部分内容。命名也按同样的方式随约定翻转：Python（按 PEP 8 用 `snake_case`）[^3] 和 C++ STL 用 `count_freqs`，Java（`camelCase`）和 Go（`MixedCaps`，首字母小写让函数成为包内私有）用 `countFreqs`。

## 速查表

四种语言，十二种惯用法。

| 惯用法 | Python | Java | C++ | Go |
|---|---|---|---|---|
| **计数器 / 频率表** | `Counter(s)` | `m.getOrDefault(c, 0) + 1` 再 `put` | `++freq[c]`（零初始化） | `freq[c]++`（零值） |
| **栈 / LIFO** | 用 `list` 的 `append`/`pop` | `ArrayDeque`（`push`/`pop`）——绝不用 `Stack`[^4] | `std::stack` 或 `vector::push_back`/`pop_back` | `[]T` 切片配 `append` 和 `s = s[:len(s)-1]` |
| **队列 / FIFO** | `collections.deque`（`popleft`） | `ArrayDeque`（`offer`/`poll`） | `std::queue`（套在 `std::deque` 上的适配器） | `[]T` 切片或环形缓冲区 |
| **小顶堆** | `heapq`（默认就是小顶） | `PriorityQueue`（默认就是小顶） | `std::priority_queue` 是 **MAX**；要小顶就传 `std::greater<>` | `container/heap`，显式写 `Less` |
| **大顶堆** | 把值取负塞进 `heapq` | `PriorityQueue<>(Comparator.reverseOrder())` | `std::priority_queue`（默认） | `container/heap`，把 `Less` 反过来 |
| **有序映射 / floor / ceiling** | `sortedcontainers.SortedDict`（第三方） | `TreeMap`（红黑树） | `std::map`（红黑树） | 标准库没有；用有序切片 + `sort.Search` |
| **遍历映射（键、值）** | `for k, v in m.items()` | `for (Map.Entry<K,V> e : m.entrySet())` | `for (auto& [k, v] : m)`（C++17） | `for k, v := range m` |
| **带下标遍历** | `for i, v in enumerate(nums)` | `for (int i = 0; i < n; i++)` | `for (size_t i = 0; i < n; ++i)` | `for i, v := range s` |
| **切片（半开区间 `[lo, hi)`）** | `s[lo:hi]`（总是复制） | `s.subList(lo, hi)`（视图） | `std::span<T>{begin+lo, hi-lo}`（C++20） | `s[lo:hi]`（与底层数组共享内存） |
| **反转序列** | `s[::-1]`（复制）或 `list.reverse()`（原地） | `Collections.reverse(list)` | `std::reverse(begin, end)` | `slices.Reverse(s)`（1.21+） |
| **带比较器排序** | `key=lambda x: (x[0], x[1])` | `Comparator.comparingInt(...).thenComparingInt(...)`[^5] | 返回 `bool` 的 lambda | `slices.SortFunc` 返回 `cmp.Compare` |
| **自定义排序，稳定** | `list.sort`（始终稳定） | `Arrays.sort(Object[])`（稳定） | `std::stable_sort` | `slices.SortStableFunc` |

陷阱层。跨语言之间最容易咬人的那些形状，也是“我本地明明跑通了”这类故事的主要来源：

| 陷阱 | Python | Java | C++ | Go |
|---|---|---|---|---|
| **整数溢出** | 没有（按 PEP 237，`int` 无上限）[^6] | 静默回绕；JLS §4.2.2：“整数运算符不以任何方式指示溢出或下溢”[^7] | 有符号溢出的行为未定义[^8]；加宽到 `long long` | 静默回绕（补码）；加宽到 `int64` |
| **默认递归深度** | 1,000（在模块顶部调到 `10**6`） | 约 5,000–20,000（由 `-Xss` 控制） | 8 MB 栈上约 50,000–200,000 | 无限制；goroutine 栈会自动增长 |
| **哈希表遍历顺序** | 插入顺序（3.7+） | `HashMap` 中未定义；要插入顺序用 `LinkedHashMap` | `unordered_map` 中未定义 | 设计上每次遍历都随机 |
| **负数输入上的 `>>`** | 向下取整（`-5 >> 1 == -3`） | 算术移位（符号扩展）；逻辑移位用 `>>>` | C++20 之前由实现定义；C++20 起为算术移位 | 有符号是算术移位；无符号是逻辑移位 |
| **排序稳定性的默认值** | 稳定 | `Object[]` 稳定，基本类型 `int[]` 不稳定 | 不稳定；用 `std::stable_sort` | 不稳定；用 `slices.SortStableFunc` |
| **比较器溢出** | 没有（任意精度） | `(a, b) -> a - b` 会溢出；得到的比较器不满足传递性，TimSort 会抛异常[^5] | 溢出时相减是 UB | 顺序错但不 panic；用 `cmp.Compare` |
| **缺失键的映射默认值** | `d[k]` 抛 `KeyError`，`d.get(k)` 返回 `None` | `get` 返回 `null`；默认值用 `getOrDefault` | 值初始化为零[^1] | `V` 的零值[^2] |
| **字符串可变性** | 不可变（要修改得先 `list(s)`） | 不可变（用 `StringBuilder`） | 可变（`std::string`） | 不可变（`string`）；要修改用 `[]byte` |
| **切片 / 视图的别名共享** | 总是复制 | `subList` 是父列表的实时视图 | 迭代器和 span 与底层容器共享内存 | 切片默认共享；要独立用 `slices.Clone` |

配套文档里的 DSH 规则就是这张表压缩成的一份强制性约定。DSH-01 禁止基于相减的比较器（即比较器溢出那一行）。DSH-04 钉死 Python 的递归上限策略。DSH-07 管四种语言的排序稳定性。DSH-08 管 `unordered_map` 在对抗性输入下的行为。

## 真正需要记住的

三个反射值得刻进肌肉记忆。它们覆盖了跨语言移植 bug 里占大头的那些失效模式。

**Python 和 Java 需要显式的查找或默认值；C++ 和 Go 不需要。** 当算法是“给这个键的计数加一”时，显式那一对用 `Counter` 或 `getOrDefault`，隐式那一对用 `++freq[k]` 或 `freq[k]++`。把 `freq[c]++` 粘进 Java 就是一个 `NullPointerException`。把 `getOrDefault` 粘进 C++ 能编译，但读起来像译文，不像原生习惯；用方括号自增的形式。

**半开区间 `[lo, hi)` 是通用约定。** 四种语言都从零开始索引，都把区间当作半开。Python 切片、Java `subList`、C++ 迭代器对、Go 切片：同一个约定。`len = hi - lo` 不需要 `+1`，这就消灭了闭区间招来的差一错误。Skiena 的 *Algorithm Design Manual* 把这条纪律讲得很明确。[^9] 本手册一律把区间写成 `[lo, hi)`，绝不写成 `[lo, hi]`。

**装箱是 Java 的税；UB 是 C++ 的税；别名共享是 Go 的税。** 每种语言都背一项另外三种能避开的固定成本。Java 的 `PriorityQueue<Integer>` 每次 `offer` 和 `poll` 都自动装箱；在 Dijkstra 的热路径上，GC 压力是可以量出来的，标准绕法是用 `int[]` 配 `Comparator.comparingInt`。C++ 的有符号溢出在 `-O2` 下是未定义行为，编译器可以自由假定它不会发生；防御办法是在相乘之前先加宽到 `int64_t`。Go 的切片共享同一个底层数组，除非你 `slices.Clone`，而 Go 章节代码里最常见的 bug 就是改了一个切片，顺手把另一个按值传进来的切片也改了。Python 一项税都不交，但交另一种：任意精度算术加上引用语义，在热路径上合起来要付三到十倍的常数因子。

## 后续衔接

[选择你的面试语言](./06-choosing-your-language.md) 是下一章，回答“给定目标公司，我该用哪种语言写”。推荐基本都是 Python；例外是 Google（只接受 Java、C++、JavaScript 或 Python）以及日常工作语言已经很顺手的候选人。

从 Part 1 开始，每一章都吃这一页的习惯规则。当[数组：静态、动态、多维](../part-1-linear-data-structures/00-arrays.md)里的 Java 标签页用 `ArrayDeque` 实现栈时，依据的就是本页的 DSH-DSA。当[哈希表](../../../dsa/part-1-linear-data-structures/03-hash-maps.md)里的 Go 标签页在向下传递之前克隆切片时，依据的就是那条别名规则。四语言姿态只有在章节和读者共用这张约定表时才成立；这张表就住在本页。

## 参考文献

[^1]: cppreference 贡献者，"std::unordered_map::operator[]"，[en.cppreference.com/w/cpp/container/unordered_map/operator_at](https://en.cppreference.com/w/cpp/container/unordered_map/operator_at)。

[^2]: The Go Authors, *The Go Programming Language Specification*, "Map types," [go.dev/ref/spec](https://go.dev/ref/spec#Map_types)

[^3]: van Rossum, G., Warsaw, B., and Coghlan, A., "PEP 8 — Style Guide for Python Code," [peps.python.org/pep-0008](https://peps.python.org/pep-0008/)。命名约定、推导式、docstring 风格。

[^4]: Oracle, *ArrayDeque (Java SE 21)*, [docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayDeque.html](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayDeque.html)

[^5]: Oracle, *Comparator (Java SE 21)*, [docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Comparator.html](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Comparator.html)

[^6]: Zadka, M. and van Rossum, G., "PEP 237 — Unifying Long Integers and Integers," [peps.python.org/pep-0237](https://peps.python.org/pep-0237/)。

[^7]: Java Language Specification, JLS 21, §4.2.2 "Integer Operations," [docs.oracle.com/javase/specs/jls/se21/html/jls-4.html](https://docs.oracle.com/javase/specs/jls/se21/html/jls-4.html#jls-4.2.2)。

[^8]: ISO/IEC 14882:2017 (C++17), §6.9.1; cppreference, "Arithmetic operators" §"Overflows," [en.cppreference.com/w/cpp/language/operator_arithmetic](https://en.cppreference.com/w/cpp/language/operator_arithmetic)

[^9]: Skiena, S.S., *The Algorithm Design Manual*, 3rd ed., Springer, 2020, ISBN 978-3-030-54256-6。把半开区间纪律当作对抗差一错误的主要工具。
