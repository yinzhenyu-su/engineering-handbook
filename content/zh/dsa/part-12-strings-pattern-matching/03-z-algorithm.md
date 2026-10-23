---
title: "Z 算法"
description: "把 Z 数组理解成字符串上的前缀匹配表；让算法保持线性的 [l, r) 窗口不变量；以及它与 KMP 在精确模式匹配上的对比。"
slug: 03-z-algorithm
part: 12
chapter: 3
difficulty: advanced
prerequisites:
  - part-12-strings-pattern-matching/02-kmp
  - part-1-linear-data-structures/02-strings
  - part-0-foundations/01-complexity-big-o
date_updated: 2026-05-25
languages: [python, java, cpp, go]
canonical_test: LC-1668
widgets: [w-40-z-array]
ladder:
  core: [LC-1668, LC-1163, LC-3008]
  stretch: [LC-3034, LC-214]
  star: LC-3008
---

# Z 算法

Gusfield 在 1997 年出版了一本 590 页的字符串算法专著，给了 Z 算法三十页篇幅，§1.5 的开篇一句话经受住了时间考验：*“我们后面要研究的几乎每一个字符串匹配问题，都可以归约成这样一个问题：对这个字符串 `s` 里的每个位置 `i`，从 `i` 开始、能与 `s` 的某个前缀匹配的子串有多长？”* 一次性、以线性时间回答每个 `i` 的这个问题，精确模式匹配就是它的推论。周期检测是推论。最长回文前缀是推论。LeetCode 路线上的五道面试题坍缩成一句话——“算一个 Z 数组，然后读它”。

[KMP 与失配函数](./02-kmp.md)用同样的时间界回答的是另一个问题。KMP 从左到右扫文本，记住失配后怎么跳；Z 则在整个字符串上建一张表，让调用方去读题目要的那几项。两个算法都是 O(n + m)，在相同输入上的渐进代价一样。两者之间怎么选基本是风格问题，本章末尾的对比表就是实用指南。本章中间那一节讲的，是两者中最难的一处构造：让 Z 保持线性的 `[l, r)` 匹配窗口。

## Z 数组的定义

对长度为 `n` 的字符串 `s`，Z 数组 `z[0..n-1]` 逐个位置定义。`z[i]` 是从下标 `i` 开始、同时又是 `s` 前缀的最长子串长度。第 0 项是特例：按约定 `z[0] = 0`，因为递推从 `i = 1` 起步，永远不读 `z[0]`。把 `z[0]` 定义成 `n` 会让推导多出一个特例，也会把循环搞乱。[^1]

拿 `s = "aabaabcaab"`（长度 10）手工跑一遍这个定义。

```text
i:    0  1  2  3  4  5  6  7  8  9
s:    a  a  b  a  a  b  c  a  a  b
z:    0  1  0  3  1  0  0  3  1  0
```

下标 3 读作：从位置 3 开始，子串 `"aab"` 与前缀 `"aab"` 匹配了 3 个字符，接着 `s[3] = 'a'` 与 `s[6] = 'c'` 失配。三次匹配，`z[3] = 3`。下标 4 读作：从位置 4 开始，单个字符 `'a'` 匹配前缀的第一个字符，接着 `s[1] = 'a'` 与 `s[5] = 'b'` 失配。一次匹配，`z[4] = 1`。下标 7 又抓到 `"aab"`，这次匹配三次后一直跑到字符串末尾；`z[7] = 3`。

这些数字里透出的规律，就是整个算法存在的理由。算完 `z[3] = 3` 之后，算法*已经*把 `s[3..5]` 与 `s[0..2]` 比过，并且知道两者相等。下标 4 和 5 就落在同一段已匹配区间里。它们的 Z 值可以直接从 `z[1]`、`z[2]` 读出来，一个字符都不用比。这就是 Z 算法兑现的那笔节省。看表：`z[4] = z[1] = 1`，`z[5] = z[2] = 0`。`z[7] = 3` 算完之后同样的把戏再来一次：`z[8] = z[1] = 1`，`z[9] = z[2] = 0`。

这是核心洞见，而且它有名字。算法最近一次确认过的匹配区间叫 **Z-box**，写成半开区间 `[l, r)`。`z[3] = 3` 之后，Z-box 是 `[3, 6)`。在这个区间内部，每个位置 `i` 都能免费知道自己的 Z 值，上限由盒子的右边界封住。

## 算法

除数组外还有两个状态变量。`l` 和 `r` 记录目前发现的最靠右的 Z-box。不变量是：`s[l:r]` 等于 `s[0:r-l]`，且在所有满足该条件的窗口里 `r` 最大。主循环分三个分支。

```python
def z_function(s):
    n = len(s)
    z = [0] * n
    l, r = 0, 0                          # 不变量：s[l:r] == s[0:r-l]
    for i in range(1, n):
        if i < r:                        # i 落在当前 Z-box 内
            z[i] = min(r - i, z[i - l])  # 取镜像值，用盒子边界封顶
        while i + z[i] < n and s[z[i]] == s[i + z[i]]:
            z[i] += 1                    # 直接比较继续扩展
        if i + z[i] > r:                 # 新匹配伸得更远
            l, r = i, i + z[i]           # 窗口整体前移
    return z
```

三个分支按顺序读。`i` 在当前盒子之外（`i >= r`）时，算法从零开始，靠直接比较扩展。`i` 在盒子之内（`i < r`）时，去看镜像位置 `k = i - l`。如果 `z[k]` 小到足以留在盒内，答案就是 `z[k]`，不需要任何字符比较。如果 `z[k]` 大到会越过 `r`，答案至少是 `r - i`（镜像里被盒子确认过的那一段），也可能更多，所以 `while` 循环从那里接着扩展。`min(r - i, z[i - l])` 这一处封顶把两种子情况写进了一个表达式。

每个位置处理完后，如果新匹配 `[i, i + z[i])` 越过了原来的 `r`，盒子就滑到这个新区间。整个算法里 `r` 只在这个滑动处前进，而且只向前进。

### 为什么内层循环是摊还线性的

`while` 循环没有显然的界。单次外层迭代里，它可能跑零次比较，也可能跑 `n` 次。但整个运行过程中，成功比较的总次数不超过 `n`。论证是同一个观察用两次。

`while` 里每次成功的字符比较都严格推进 `r`。看循环体：每匹配一次，`z[i]` 加一；到外层迭代结束时，滑动条件 `i + z[i] > r` 成立，把 `r` 置为 `i + z[i]`。所以一次匹配至少让 `r` 涨一。整个过程中 `r` 从 0 出发，永远不超过 `n`，因此总共最多涨 `n`。成功比较的次数于是最多 `n` 次。[^2]

每次外层迭代在平凡初始化或镜像步骤之后，最多再做一次*失败*比较，就是让 `while` 停下的那一次。外层迭代有 `n - 1` 次，最多贡献 `n - 1` 次失败比较。合计：整个算法最多 `2n - 1` 次字符比较，即 O(n)。[^1]

这和把 KMP 失配函数限制在线性时间的是同一个势能论证：一个只朝一个方向走的计数器，在整个运行上求和。Z 算法与 KMP 共享这种证明形态，因为两者共享同一个底层思路——从先前的匹配里继承工作；区别只在于哪个计数器当主角、朝哪个方向走。

## 实例推演：LC 1668

看 Z 算法凭什么站得住，最干净的地方是一道暴力解法几乎能过的题。**LC 1668（Maximum Repeating Substring）** 问：给定字符串 `sequence` 和 `word`，返回最大的 `k`，使 `word` 重复 `k` 次后作为子串出现在 `sequence` 里。[^3] 样例输入 `sequence = "ababc"`、`word = "ab"` 的答案是 2，因为 `"abab"` 是子串，`"ababab"` 不是。

最自然的第一版写法是枚举 `sequence` 的每个起点，用 `String.contains` 或等价物检查 `word * k` 是否出现。结果对，在小约束规模下也能进 LC 的时限，但渐进上界是 `O(|sequence|² × |word|)`，而且工作的结构恰恰是 Z 要修掉的那种浪费：同一份前缀匹配检查，在同一批下标上被做了成千上万次。

Z 算法把它换成一次预处理加一次廉价扫描。构造 `s = word + "#" + sequence`，哨兵字符在两个输入里都不出现。算 `z(s)`。`s` 里哨兵之后每个满足 `z[i] >= |word|` 的位置 `i`，就是 `word` 在 `sequence` 中出现的一个位置。要从某个起点数重复次数，以 `|word|` 为步长往前走，数连续命中的段长。步长 `|word|` 上连续 `z[i] >= |word|` 的最长一段，就是答案。

```python
from typing import List


def z_function(s: str) -> List[int]:
    n = len(s)
    z = [0] * n
    l, r = 0, 0
    for i in range(1, n):
        if i < r:
            z[i] = min(r - i, z[i - l])
        while i + z[i] < n and s[z[i]] == s[i + z[i]]:
            z[i] += 1
        if i + z[i] > r:
            l, r = i, i + z[i]
    return z


def max_repeating(sequence: str, word: str) -> int:
    m = len(word)
    if m == 0 or m > len(sequence):
        return 0
    s = word + "#" + sequence
    z = z_function(s)
    best, n = 0, len(sequence)
    for start in range(n):
        i = m + 1 + start
        run = 0
        while i + m <= len(s) and z[i] >= m:
            run += 1
            i += m
        best = max(best, run)
    return best
```

**解法** —— Python（[Java](./03-z-algorithm/lc-1668/sol.java) · [C++](./03-z-algorithm/lc-1668/sol.cpp) · [Go](./03-z-algorithm/lc-1668/sol.go)）

```python
# LC 1668. Maximum Repeating Substring
from typing import List


def z_function(s: str) -> List[int]:
    """Return the Z-array of s in O(n) time. z[0] = 0 by convention."""
    n = len(s)
    z = [0] * n
    l, r = 0, 0
    for i in range(1, n):
        if i < r:
            z[i] = min(r - i, z[i - l])
        while i + z[i] < n and s[z[i]] == s[i + z[i]]:
            z[i] += 1
        if i + z[i] > r:
            l, r = i, i + z[i]
    return z


def max_repeating(sequence: str, word: str) -> int:
    """LC 1668: largest k such that word repeated k times is a substring of sequence."""
    m = len(word)
    if m == 0 or m > len(sequence):
        return 0
    s = word + "#" + sequence
    z = z_function(s)
    # 在文本部分上走 Z 数组，以步长 m 串联匹配
    best = 0
    n = len(sequence)
    for start in range(n):
        i = m + 1 + start
        run = 0
        while i + m <= len(s) and z[i] >= m:
            run += 1
            i += m
        if run > best:
            best = run
    return best
```

那个 `+ "#" +` 很关键，也是初学者写 Z 时最常漏掉的一处细节。没有哨兵，文本里的前缀匹配会溢出到模式的后缀匹配上，得出的 Z 值混着两个边界。有了哨兵，任何 Z 值都越不过它；模式和文本彼此封死。挑一个在两个输入里都不出现的字符即可。对 LC 的小写字母约束来说，`#` 是通用选择。[^2]

> **Interactive widget:** [Z 数组构造与 [l, r) 匹配窗口](../../../dsa/widgets/w-40-z-array.yml)（面板 `default`）—— 在网站上渲染；YAML 规格描述了各个关键帧。

*静态回退：在 `s = "aabaabcaab"` 上，Z 算法共走 19 步。第 8 步算出 `z[3] = 3` 后，窗口滑到 `[3, 6)`；第 15 步之后又滑到 `[7, 10)`。第一次复用镜像是第 9 步：`z[4] = z[1] = 1`，一个字符都没比较，因为第 8 步的滑动已经证明了 `s[3..5]` 等于 `s[0..2]`。最终数组：`[0, 1, 0, 3, 1, 0, 0, 3, 1, 0]`。下标 3 和下标 7 这两个非零位置，就是前缀 `"aab"` 的再次出现。*

## 实际开销

| 变体 | 时间 | 空间 | 说明 |
|---|---|---|---|
| Z 数组构造 | O(n) | O(n) | 在 `"aaaa..."` 这类对抗输入上依然很紧 |
| 模式匹配 `pattern + # + text` | O(n + m) | O(n + m) | 与 KMP 的界等价 |
| 从已建好的 Z 数组做周期检测 | O(n) 额外 | O(1) 额外 | 在现有数组上扫一遍线性 |
| 朴素子串搜索（基线） | O(n × m) | O(1) | 在 LC 3008 的 `n = 10^5`、`m = 10^5` 上超时 |

这个界的成立*与字母表大小无关*。对字符唯一的操作是 `==`，所以算法在 ASCII、Unicode 码点，或任何带相等谓词的对象上跑得一样。KMP 也有这个性质；Boyer-Moore 没有，这正是它在小字母表上常常吃亏、哪怕在大字母表上更占优的原因。[^4]

空间上界是 Z 数组自身的 O(n)；`[l, r)` 状态只是两个整数。全文没有 `O(n × m)` 的矩阵——Z 和 KMP 一样，是披着表的*流式*算法。

## 模式失效的场景：三个值得知道的归约

模式匹配是它的标准应用，但同一个原语还能回答几个表面毫无关系的问题。

**周期检测。** 满足 `i + z[i] == n` 且 `i` 整除 `n` 的最小 `i > 0`，就是 `s` 的周期。若 `s = "abcabcabc"`，则 `z[3] = 6`，`3 + 6 = 9 = n`，且 `3` 整除 `9`，周期为 `3`。[^2] LC 459（Repeated Substring Pattern）由此直接得解：答案当且仅当存在这样的 `i`。数组建成之后，Z 数组以 O(n) 的额外代价给出周期。

**最长回文前缀。** 构造 `s' = s + # + reverse(s)`，算 `z(s')`，找满足 `z[i] + i == |s'|` 的最大 `i`。后半段的这个匹配位置，对应 `s` 中与自身反转相等的最长前缀，也就是最长回文前缀。[^5] LC 214（Shortest Palindrome）归约到这一步：把最长回文前缀之后的所有内容反转后前置，得到的就是以 `s` 为前缀的最短回文串。

**整数数组上的模式匹配。** Z 算法的逻辑只依赖两个元素是否相等，不依赖字符串专有操作。LC 3034（Number of Subarrays That Match a Pattern I）用成对比较把整数数组转成 `(+1, 0, -1)` 方向码数组，再在码上跑 Z，以 O(n + m) 找出所有匹配目标序列的子数组。[^6] 同一份代码对任何定义了 `==` 的序列都成立。

三者共通的规律：如果问题能改写成*“这个前缀在字符串内部哪里再次出现？”*这个形式，Z 就合适。上面三个归约把看起来差得很远的问题改写成那个形状。

## 两个最容易踩的坑

> [!WARNING]
> **忘了给镜像值封顶。** 写成 `z[i] = z[i - l]` 而不是 `z[i] = min(r - i, z[i - l])`，镜像就会把一个越过 `r` 的值抄过来，而那段区间算法并没有真正验证过。于是算法声称 `z[i]` 在未验证的位置上匹配，在 `"aaaabaa"` 这类输入的 `i = 6` 处给出错误答案。[^2] 这个封顶是承重的正确性步骤。永远写 `min`。

> [!WARNING]
> **半开 `[l, r)` 与闭区间 `[L, R]` 之间的差一。** 最早那篇 Codeforces 教程两种约定混着用：正文是闭区间，代码是半开，每个分支末尾还来一句 `R--`。[^7] 照正文抄进半开区间代码库的实现会写成 `i <= r` 而不是 `i < r`，让镜像分支在镜像值未定义的边界上触发。选定一种约定。上面用的是 cp-algorithms.com 的半开约定，判断 `i < r`，滑动写 `r = i + z[i]`。在所有地方都跟它对齐——伪代码、四种语言的样例、以及任何对算法的讲述。混用会产生一个静默的差一错误，代价是一场面试。

还有第三个值得点一句的坑：因为整串平凡地匹配自身，就把 `z[0]` 定义成 `n`。递推在 `i < r` 时会读 `z[i - l]`；如果 `i - l` 恰好等于 0，读到的就是 `z[0]`。把 `z[0]` 设成 `n` 会污染镜像初始化。设 `z[0] = 0`；算法不会为别的目的读它。

## KMP 与 Z：各自何时选用

KMP 和 Z 在相同的时间界内算出不同、但可以互相转换的值。[^8] 在纯模式匹配题（LC 28、LC 459）上，问“线性时间字符串匹配”的面试官两种都接受，多数 LC 题解也把两者都列出。选择主要看题目天然要的是哪张*表*。

| 关注点 | KMP（12.2 章） | Z 算法（本章） |
|---|---|---|
| 模式匹配时间 | O(n + m) | O(n + m) |
| 空间 | 在模式上建失配函数，O(m) | 在 `pattern + # + text` 上建 Z 数组，O(n + m) |
| 天然回答什么 | “把文本扫一遍，失配后跳过” | “每个位置上的前缀匹配有多长？” |
| 在长度未知的文本上流式处理 | 直接可行（失配函数只为模式建一次） | 需要先拿全文本（Z 建在拼接串上） |
| 周期检测 | 直接（失配函数暴露周期） | 直接（Z 值加下标） |
| 最长回文前缀 | 可用反转技巧做到 | 用 `z(s + # + reverse(s))` 自然得到 |
| 在带 `==` 的任意对象上做模式匹配 | 可行 | 可行 |
| 心智模型 | 一台边扫边算的状态机 | 一张由调用方读取的表 |
| 凭记忆上白板 | 多数读者能画出失配函数 | 不少读者会在 `[l, r)` 封顶处翻车 |
| 历史上的主导地位 | 自 1977 年起的教科书标准 | 自 2000 年代初起的竞赛编程标准 |

要记住的不对称：**KMP 沿着流扩展，Z 沿着表扩展。** 当问题是“在这段文本里找一次这个模式”、而文本一个字符一个字符到达时，KMP 天然合适。当问题是“告诉我这个字符串的前缀与其余部分如何交互的一切”——不管那是周期、回文前缀，还是所有匹配位置——Z 天然合适，因为前缀匹配表正是你要的东西。

LC 3008 是本章的招牌题，也是一个干净的例证。任务是对两个模式 `a`、`b` 和一段文本 `s`，找出 `s` 中所有 `a` 出现、且 `b` 也出现在距离 `k` 以内的下标。解法分别算出 `z(a + # + s)` 与 `z(b + # + s)`，取出两个有序下标列表，再用双指针或二分扫描处理距离约束。KMP 也能解，把失配函数扫描跑两遍，代价相同。题解实现偏向 Z 的原因是：题目天然的诉求“每一个匹配位置”*就是*一张表，而 Z 就是这张表。

关于对比的诚实收尾：自学字符串匹配时先学 KMP，题目直接要前缀匹配表时先用 Z。两个都带着。它们不是对手，而是同一个问题透过不同镜片看到的样子；哪片镜片贴合问题，就用哪片。

## 题目梯度

### 核心（三题全做才算掌握本章）

- [LC 1668 — Maximum Repeating Substring](https://leetcode.com/problems/maximum-repeating-substring/) [简单] • z-algorithm
- [LC 1163 — Last Substring in Lexicographical Order](https://leetcode.com/problems/last-substring-in-lexicographical-order/) [困难] • z-algorithm *（参考实现：仅 Python）*
- [LC 3008 — Find Beautiful Indices in the Given Array II](https://leetcode.com/problems/find-beautiful-indices-in-the-given-array-ii/) [困难] • z-algorithm ★ *（参考实现：仅 Python）*

### 进阶（可选）

- [LC 3034 — Number of Subarrays That Match a Pattern I](https://leetcode.com/problems/number-of-subarrays-that-match-a-pattern-i/) [中等] • z-algorithm *（参考实现：仅 Python）*
- [LC 214 — Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome/) [困难] • z-algorithm *（参考实现：仅 Python）*

LC 28（Find the Index of the First Occurrence in a String）是标准的“线性时间子串搜索”题，故意没放进本题梯度。12.0 章已把它作为核心题收走；那里的框架——Z 用 `z(pattern + # + text)` 替代 KMP 得到 O(n + m) 解——是上面实例推演的推论。LC 459（Repeated Substring Pattern）由 [KMP 与失配函数](./02-kmp.md) 收走，它也能从周期归约得到 Z 解法。

LC 3034 放在进阶里，充当本章 E/M/H 跨度中的 M 位；要学的是算法迁移（字符换成整数码）。LC 214 是题解里得到回报的反转拼接技巧；把它列进来，也在强调同一个原语能回答乍看与字符串匹配无关的回文问题。

## 后续衔接

Z 数组与 KMP 的失配函数是两个基础的线性时间字符串匹配原语，二者都能自然延伸到更丰富的问题。[多模式匹配的 Trie（Aho-Corasick 直觉）](./04-aho-corasick.md) 把 KMP 的失配链接思路搬进 Trie，把线性时间匹配从一个模式抬到多个模式。[后缀数组](../../../dsa/part-12-strings-pattern-matching/05-suffix-array.md) 更进一步：它不回答“*这个*前缀出现在哪里”，而是回答“`s` 的*任意*子串出现在哪里”，构造代价 O(n log n)；其中若干构造（Karkkainen-Sanders、DC3）把 Z 数组式的摊还论证当作积木。

另一条分支横向延伸。[Rabin-Karp](./01-rabin-karp.md) 是第三个线性时间字符串匹配算法，权衡画像不同：期望 O(n + m)，最坏 O(n × m)，外加一个实用优势——一个滚动哈希就能推广到多模式匹配和二维匹配。本章所属的模式决策页把 KMP、Z、Rabin-Karp 并排摆开；本章是这三者里的中间那位。

## 参考文献

[^1]: Dan Gusfield, *Algorithms on Strings, Trees, and Sequences: Computer Science and Computational Biology*, Cambridge University Press, 1997.

[^2]: cp-algorithms.com, "Z-function and its calculation." <https://cp-algorithms.com/string/z-function.html>.

[^3]: walkccc.me, "1668. Maximum Repeating Substring." <https://walkccc.me/LeetCode/problems/1668/>.

[^4]: Jeff Erickson, *Algorithms*（在线教材），第 6 章 §6.2 "String Matching"。把前缀函数类方法（KMP、Z）放在同一个摊还窗口论证下处理，并指出两者区别于 Boyer-Moore 的字母表无关性。<https://jeffe.cs.illinois.edu/teaching/algorithms/>.

[^5]: leetcodethehardway.com, "0214 - Shortest Palindrome (Hard)." <https://leetcodethehardway.com/solutions/0200-0299/shortest-palindrome-hard>.

[^6]: walkccc.me, "3034. Number of Subarrays That Match a Pattern I." <https://walkccc.me/LeetCode/problems/3034/>.

[^7]: paladin8, "Z Algorithm." Codeforces blog entry 3107, 2010. <https://codeforces.com/blog/entry/3107>.

[^8]: cp-algorithms.com, "Prefix function — Knuth-Morris-Pratt." <https://cp-algorithms.com/string/prefix-function.html>.
