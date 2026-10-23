---
title: "朴素字符串匹配"
description: "藏在 str.find 与 str.indexOf 背后的 O(nm) 双重循环。它在真实文本上的平均表现，以及那些逼出 KMP 与 Rabin-Karp 的对抗性输入。"
slug: 00-naive-string-matching
part: 12
chapter: 0
difficulty: intermediate
prerequisites:
  - part-1-linear-data-structures/02-strings
  - part-0-foundations/01-complexity-big-o
date_updated: 2026-05-25
languages: [python, java, cpp, go]
canonical_test: LC-28
widgets: []
ladder:
  core: [LC-28, LC-1455, LC-686]
  stretch: [LC-30, LC-796]
  star: LC-28
---

# 朴素字符串匹配

在 Python 里敲下 `"hello world".find("world")`，得到 `6`。同样的调用在 Java 里是 `"hello world".indexOf("world")`。C 的 `strstr("hello world", "world")` 返回一个指针，指向 `"world"` 起点之后四个字节的位置。Go 的 `strings.Index("hello world", "world")` 同样返回 `6`。四种语言，四种写法，干的是同一件事：告诉我模式串（needle）从哪儿开始。

这些调用你用过不下一千次。它们太不起眼，以至于很少有工程师能说清背后到底在干什么。本章就来回答这个问题。函数名底下那套算法短到能写在明信片上，在真实文本上跑得足够快、没人抱怨，可一旦碰上对抗性输入就崩得很难看——第 12 部分后三章的存在，主要就是为了修它。

## 标准库就是这套算法

每个现代运行时都自带子串匹配。没有哪个用了什么奇技淫巧。Python 的 `str.find` 干的差不多就是接下来要写的东西；CPython 的实现围着它加了几个常数级的小技巧，形状还是同一个。Java 的 `String.indexOf` 就是教科书上的双重循环，平铺直叙，没有任何修饰。C++ 的 `std::string::find` 也一样。Go 的 `strings.Index` 对短模式串走短路，长一些的转投 Rabin-Karp 变体，但 32 字节以内的模式串用的仍是朴素字节比较。

例外是现代 C 的 `strstr`。glibc 和 musl 都在 2000 年代的某个时候，把朴素的 `strstr` 换成了 Crochemore-Perrin two-way 算法，最坏情况 `O(n)`，额外空间 `O(log m)`。[^glibc] 所以在多数系统上，`strstr` *并不*是本章讲的东西。它是 `strstr` 曾经的样子，也是 `String.indexOf` 现在的样子。

为什么要教一个 C 库已经弃用的版本？两个理由。朴素算法是基线，12.1（Rabin-Karp）、12.2（KMP）、12.3（Z）各自在它之上改进；如果没见过被替换掉的那个算法，读这三章就像在看魔术。而且在朴素算法跑得动的那些输入上（随机文本、英文散文、任何没有被刻意构造的对抗样本），它的平均复杂度是 `Θ(n + m)`，[^wiki-kmp] 与 KMP 同一个界，还不用维护失败表。标准库把它留了那么久，就是因为对真实输入来说它够用。

## 五行写出的朴素匹配器

模式就是一个直白的双重循环。对主串里的每个起始位置 `i`，拿模式串的至多 `m` 个字符，跟主串从 `i` 开始的部分逐一比较。`m` 个全部相同就返回 `i`；只要有一个不同，`i` 前进一位再来一遍。没有预处理，没有位移表，没有滚动状态。

```python
# LC 28 — Find the Index of the First Occurrence in a String
def str_str(haystack: str, needle: str) -> int:
    n, m = len(haystack), len(needle)
    if m == 0:
        return 0
    if m > n:
        return -1
    for i in range(n - m + 1):
        j = 0
        while j < m and haystack[i + j] == needle[j]:
            j += 1
        if j == m:
            return i
    return -1
```

**解法** — Python（[Java](./00-naive-string-matching/lc-28/sol.java) · [C++](./00-naive-string-matching/lc-28/sol.cpp) · [Go](./00-naive-string-matching/lc-28/sol.go)）

```python
# LC 28. Find the Index of the First Occurrence in a String
def str_str(haystack: str, needle: str) -> int:
    """Return the first index where needle occurs in haystack, or -1."""
    n, m = len(haystack), len(needle)
    if m == 0:
        return 0
    if m > n:
        return -1
    for i in range(n - m + 1):
        j = 0
        while j < m and haystack[i + j] == needle[j]:
            j += 1
        if j == m:
            return i
    return -1
```

两个守卫在做实事，还有一个循环边界值得留意。空模式串返回 `0`，因为所有 `find` 的实现都把“空模式串在 0 处匹配”定为真（Python 里 `"abc".find("") == 0`，Java 一样，C 也一样）。模式串比主串长时返回 `-1`，因为怎么挪都塞不下。外层循环从 `0` 跑到 `n - m`，**两端都含**：Python 里写 `range(n - m + 1)`，Java、C++、Go 里写 `i <= n - m`。这里的差一错误是真会踩的。把 `+ 1` 去掉，那些正好从主串末尾开始的匹配（`i = n - m`）就被漏掉了。

CLRS 把这套算法印作 `NAIVE-STRING-MATCHER`，并指出它“效率低下，因为对某个 s 值取得的文本信息，在考虑其他 s 值时被完全忽略了”。[^clrs] Crochemore、Hancart 和 Lecroq 把它印作“Brute Force”，内层的相等判断展开成上面那样的逐字符循环。[^chl] 两者除了下标约定之外完全一致。

## 催生第 12 部分的最坏情况

拿主串 `"aaaaaaaa"`（八个 `a`）和模式串 `"aaab"`。匹配器检查位置 0：`a == a`、`a == a`、`a == a`，然后 `a != b`。不匹配。滑到位置 1，检查 `a == a`、`a == a`、`a == a`，然后 `a != b`。还是不匹配。同样差一点就成的戏码在每个起始位置重演，函数最终在烧掉 `(n - m + 1) · m` 次比较之后返回 `-1`。

维基百科的 KMP 词条把这个病态例子讲得很漂亮：想象主串是一百万个 `A`，模式串是 999 个 `A` 后面跟一个 `B`。朴素算法要在大约十亿次字符比较之后，才判定模式串不在里面。[^wiki-kmp] 按每秒几亿次简单操作算，这在手机上是能感觉到的等待，在服务器上是不可接受的等待。

这个形状可以推广。任何以不匹配字符收尾、其余部分却由一长串匹配前缀搭起来的模式，都会触发同样的爆炸。从“这次位移上前 999 个字符都匹配”里拿到的信息，正是 KMP 保留、朴素算法丢掉的信息。朴素算法每次位移都把内层循环从 `j = 0` 重新开始；后面三章各自找到一种办法，复用已经比较过的结果。

## 开销

令 `n = |haystack|`、`m = |needle|`。外层循环跑 `n - m + 1` 次，内层循环每次外层迭代最多跑 `m` 次，相乘就是最坏情况的界。算法不用任何辅助数据（没有哈希，没有失败表，没有 Z 数组），所以空间是常数。

| 情形 | 时间 | 空间 |
|---|---|---|
| 最好情况 | `O(n)` | `O(1)` |
| 平均情况（随机文本，字符集大小 σ） | `Θ(n + m)` | `O(1)` |
| 最坏情况 | `O(n · m)` | `O(1)` |

平均情况的界值得给个证明梗概。对字符集大小为 σ 的均匀随机文本，任一给定位移上、首次失配之前的期望比较次数是几何均值 `< σ / (σ - 1)`。σ = 26（英文字母）时约等于每次位移 1.04 次比较，于是总工作量是 `(n - m + 1) · 1.04 ≈ n`，再加上真有匹配那次位移的 `m` 次比较。合计 `Θ(n + m)`，与 KMP 同一个界。[^wiki-kmp] 算 KMP 失败表的那点常数开销，正是朴素匹配在真实文本上还能打的原因。

空间那一列是这套算法相对第 12 部分其余内容唯一的结构性优势。Rabin-Karp 只多 `O(1)` 的滚动哈希状态，但哈希撞上之后还得逐字验证。KMP 多一张 `O(m)` 的失败表。Z 算法要算一个 `O(n + m)` 的 Z 数组。当瓶颈是内存而不是时间时（嵌入式系统、紧循环里嵌着的小查找、任何“分配一张表比跑那些比较还贵”的场合），朴素算法赢。

## 模式失效的场景

题目梯度里有三道题，用的都是同一个五行匹配器，只是在外面套了不同的壳。

**找出所有出现位置，而不只是第一处。** 把 `return i` 换成 `result.append(i)`，然后越过这次匹配继续往右走。位置每次加一（重叠的也算）或加 `m`（不算重叠），按题目要求选。CLRS 的伪代码默认就是全出现位置那种形式；只找第一处是特化，不是自然形态。

**在构造出来的主串上搜索。** [LC 686 — Repeated String Match](https://leetcode.com/problems/repeated-string-match/) 要求把字符串 `a` 至少重复多少次，`b` 才会出现在里面。把 `a` 拼接正好 `ceil(|b| / |a|)` 次，跑朴素搜索，找到就返回这个次数。没找到就再补一份，再搜一次。还没有就返回 `-1`。这个 `+1` 的上界是可证明紧的：再拼更长也没用，因为 `b` 的任何一次匹配都必须从某一轮重复的前 `|a|` 个字符之内开始。算法是朴素匹配，巧的是“该造多长的主串”这个界。

**用加倍文本的技巧。** [LC 796 — Rotate String](https://leetcode.com/problems/rotate-string/) 问一个字符串是不是另一个的旋转。`s` 的每一种旋转都作为子串出现在 `s + s` 里。一行解法是 `len(s) == len(goal) and goal in s + s`，它对加倍后的文本调用朴素匹配（或者运行时实际用的那套）。长度守卫防止两串长度不同时误判为真，剩下的交给旋转定理。

## 两个最容易踩的坑

> [!WARNING]
> **部分匹配之后按 `j` 滑动。** 在位移 `i` 处匹配了 `k` 个字符、第 `k+1` 个失配之后，人很容易写成 `i += k` 而不是 `i += 1`，理由是“前 k 个已经匹配上了，可以跳过”。这*确实*是 KMP 会做的优化，但前提是有失败函数兜着。没有它，只要模式串存在某个既是已匹配部分的后缀、又真是自身前缀的非平凡前缀，这个跳法就会漏掉匹配。朴素匹配里永远是 `i += 1`。忍住这个优化，等你读到 [KMP](./02-kmp.md)——那一章正是让这个优化变正确的地方。

> [!WARNING]
> **在内层循环里做切片。** 在 Python 里写 `if haystack[i:i+m] == needle:`，或者在 Java 里写 `haystack.substring(i, i+m).equals(needle)`，读起来很自然，但每次位移都新分配一个字符串。比较仍然是 `O(m)` 时间，可现在每次位移还要 `O(m)` 额外空间，总分配数是 `n - m + 1` 个字符串，全得等垃圾回收器来收。参考实现里的显式字符循环做同样的比较，一次分配都没有。在 LC 28 的测试集上切片写法能过；上 profiler 立刻就露馅。切片写法留给原型，生产代码里老老实实写字符循环。

## 题目梯度

### 核心（三题全做才算掌握本章）

- [LC 28 — Find the Index of the First Occurrence in a String](https://leetcode.com/problems/find-the-index-of-the-first-occurrence-in-a-string/) [简单] • naive-string-matching ★
- [LC 1455 — Check If a Word Occurs As a Prefix of Any Word in a Sentence](https://leetcode.com/problems/check-if-a-word-occurs-as-a-prefix-of-any-word-in-a-sentence/) [简单] • naive-string-matching *（参考实现：仅 Python）*
- [LC 686 — Repeated String Match](https://leetcode.com/problems/repeated-string-match/) [中等] • naive-string-matching *（参考实现：仅 Python）*

### 进阶（可选）

- [LC 30 — Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/) [困难] • naive-string-matching + hash-counter *（标准解法在 [滑动窗口：固定长度](../part-3-pointers-window-prefix/02-sliding-window-fixed.md)）*
- [LC 796 — Rotate String](https://leetcode.com/problems/rotate-string/) [简单] • naive-string-matching *（参考实现：仅 Python）*

这个模式本身没有对应的标准 Hard 题。LeetCode 上打了朴素匹配标签的那几道 Hard（LC 214 Shortest Palindrome、LC 1392 Longest Happy Prefix），标准解法都是 KMP 或 Z 算法，归那两章管。LC 30 的暴力枚举*就是*在每个起始下标上做朴素匹配，所以它在这里列为进阶，而最优解归 [滑动窗口：固定长度](../part-3-pointers-window-prefix/02-sliding-window-fixed.md) 那一章。

## 后续衔接

[Rabin-Karp 与滚动哈希](./01-rabin-karp.md) 在位移过程中为当前文本窗口维护一个滚动哈希；朴素算法 `O(n · m)` 的最坏情况与 `Θ(n)` 的平均情况之间的差距，在哈希一致的那一刻就弥合了——哈希是否一致用 `O(1)` 判定，只有真的对上了才付全价去验证。[KMP 与失败函数](./02-kmp.md) 把“最长既真前缀又真后缀”的信息跨位移保留下来，在不牺牲简单性的前提下把最坏情况从 `O(n · m)` 变成 `Θ(n + m)`。[^kmp1977] [Z 算法](./03-z-algorithm.md) 通过另一种往往更好讲的数据结构达到同样的 `Θ(n + m)` 界。每一个都在回答朴素匹配答不出的那个问题：*我们此前已经知道了什么，下次怎么用上？*

## 参考文献

[^clrs]: Cormen, Leiserson, Rivest, Stein. *Introduction to Algorithms*, 4th ed., MIT Press, 2022. §32.1 "The naive string-matching algorithm." 过程 `NAIVE-STRING-MATCHER(T, P)`，循环边界 `for s = 0 to n - m`；最坏情况的界 `O((n - m + 1) m)`。

[^chl]: Crochemore, Hancart, Lecroq. *Algorithms on Strings*, Cambridge University Press, 2007. §1.2 "Brute force algorithm." 配套站点 [monge.univ-mlv.fr/~lecroq/string](https://web.archive.org/web/20241217120010/http://monge.univ-mlv.fr/~lecroq/string/)，给出标准伪代码与最坏情况分析。

[^wiki-kmp]: Wikipedia, "Knuth-Morris-Pratt algorithm." [en.wikipedia.org/wiki/Knuth-Morris-Pratt_algorithm](https://en.wikipedia.org/wiki/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm). 其中 "Background" 一节给出那对对抗样例（`s = "AAAA...A"`，长度 1M，`w = 999 As followed by B`）与由此得到的十亿次比较最坏情况，以及均匀随机文本下 `Θ(n)` 的平均情况论证。

[^glibc]: glibc, `string/str-two-way.h`。Crochemore-Perrin two-way 字符串匹配算法是 GNU C 库中朴素 `strstr` / `memmem` 的生产级替代。two-way 最坏情况 `O(n)`、额外空间 `O(log m)`，既压过朴素算法 `O(n · m)` 的最坏情况，又保留了“不需要预处理表”的手感。源码：[code.woboq.org/userspace/glibc/string/str-two-way.h](https://code.woboq.org/userspace/glibc/string/str-two-way.h.html)。

[^kmp1977]: Knuth, D. E.; Morris, J. H.; Pratt, V. R. "Fast pattern matching in strings." *SIAM Journal on Computing* 6 (2), pp. 323-350, 1977. [doi.org/10.1137/0206024](https://doi.org/10.1137/0206024)。这篇原始论文在模式匹配上击败了朴素算法的 `O(n · m)`，也是“为什么还需要一篇基线章节”这件事的历史原因。
