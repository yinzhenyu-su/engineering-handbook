---
lc_id: 3
title: "无重复字符的最长子串"
slug: LC-003-longest-substring-without-repeating-characters
primary_chapter: part-3-pointers-window-prefix/03-sliding-window-variable
star: true
difficulty: medium
date_updated: 2026-05-25
languages: [python, java, cpp, go]
editorial_widget: e-LC003-longest-substring
---

# 无重复字符的最长子串

LC 3 是可变长度滑动窗口最标准的入门题。[滑动窗口：可变长度](../part-3-pointers-window-prefix/03-sliding-window-variable.md) 里的招牌题就是这道，而这里要讲的道理只有一句：当窗口必须收缩才能恢复某个不变量时，你要么让左指针一次挪一个字符，要么维护一张「上次出现位置」表，用 O(1) 直接跳过去。收缩循环教你认清这个模式，跳跃版本才是该交上去的那份。

用来诊断的输入是 `\"pwwkew\"`，答案是 3。有意思的地方不在于最优解是 `\"wke\"`，而在于算法必须在走到中途时*忘掉*开头那个 `'w'`，而后面第二个 `'w'` 又逼它再忘一次。任何把重复字符当成终止条件、而不是窗口收缩触发器的解法，都会在这个输入上翻车。

## 题目

给定字符串 `s`，返回 `s` 中最长的、不含重复字符的子串长度。子串是连续的一段；有多个同样长的答案无所谓，因为要返回的是整数长度，不是那一段本身。约束是 `0 <= s.length <= 5 * 10^4`，`s` 取自英文字母、数字、符号和空格（可打印 ASCII 子集，128 个码点）。[^1]

| 输入 | 期望 | 考察什么 |
|---|---|---|
| `\"abcabcbb\"` | `3` | 三个并列最优（`\"abc\"`、`\"bca\"`、`\"cab\"`）；无花招的标准情形 |
| `\"bbbbb\"` | `1` | 第一步之后每一步都把 `l` 往前跳一格；窗口始终长度为 1 |
| `\"pwwkew\"` | `3` | 六个字符里两次因重复而跳跃；诊断用例 |
| `\"abba\"` | `2` | 陈旧记录的陷阱；抓 bug 的用例 |

最后这一行，就是大多数「差一点就对」的提交栽倒的地方。记住它。原文措辞见 [LC 3 Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)。

## 解法 1：暴力

照着题意直译。枚举每个起点 `i`，从 `i` 开始向后扩 `j`，只要还没出现重复就继续，记下最长的一段。

```python
# 暴力：O(n^2) —— 我们要替换掉的东西
def length_of_longest_substring_brute(s):
    best = 0
    for i in range(len(s)):
        seen = set()
        for j in range(i, len(s)):
            if s[j] in seen:
                break
            seen.add(s[j])
            best = max(best, j - i + 1)
    return best
```

每个起点都会一直走到撞上重复字符为止，所以最坏情况大约要做 `n^2 / 2` 次字符检查。当 `n = 5 * 10^4` 时就是约 `1.25 * 10^9` 次操作，在大测试点上超出 LeetCode 的 1000 毫秒预算。[^1] 这份代码在每个样例上都对，在隐藏用例上必定 TLE。把「这是 O(n²)，还能更好」这句话说出口，就是下一步所需要的校准。

## 解法 2：收缩循环滑动窗口

只要盯着暴力解法跑一遍，冗余立刻就能看见。对 `\"pwwkew\"` 从 `i = 0` 出发，内层扫过 `p, w, w` 就退出；从 `i = 1` 出发，扫过 `w, w` 就退出；从 `i = 2` 出发，扫过 `w, k, e, w` 又退出。内层循环在*重新扫描*那些前一个起点已经证明过不重复的字符。

一个窗口，两个指针，再加一份状态，用来记住暴力解法忘掉的东西。

```python
# 收缩循环：均摊 O(n)，但单步最坏是 O(n)
def length_of_longest_substring_shrink(s):
    counts = {}
    l = 0
    best = 0
    for r, ch in enumerate(s):
        counts[ch] = counts.get(ch, 0) + 1
        while counts[ch] > 1:
            counts[s[l]] -= 1
            l += 1
        if r - l + 1 > best:
            best = r - l + 1
    return best
```

外层 `for` 每处理一个字符就把 `r` 前进一格。内层 `while` 在新字符让它的计数超过 1 时收缩 `l`，一旦降回 1 就停。按教科书上的聚合分析，均摊代价是 `O(n)`：每个字符进窗口一次、出窗口至多一次，所以指针移动总量不超过 `2n`。[^2]

麻烦在于*单步*代价。遇到病态输入——比如 5 万个互不相同的字符，后面接一个与首字符相同的字符——内层循环会在 `r` 前进一格的过程中把整个窗口一个字符一个字符地放干。均摊 `O(n)`，单步最坏 `O(n)`。总量仍然落在 `O(n)`，但常数因子大约是最优解的两倍；而在真正对抗性的序列上，LC 3 的 LeetCode 讨论帖确实记录过收缩循环的提交超时、跳跃版本不超时的情况。[^3]

## 解法 3：last-index 跳跃（最优）

当新字符上次出现在下标 `prev` 时，`l` 的目的地是 `O(1)` 就能知道的：就是 `prev + 1`。收缩循环一路走过去，这里把它换成一张 `last_index` 表加一条赋值语句。

```python
def length_of_longest_substring(s):
    last = {}
    l = 0
    best = 0
    for r, ch in enumerate(s):
        if ch in last and last[ch] >= l:    # 守卫条件是必需的
            l = last[ch] + 1                # O(1) 跳跃，没有收缩循环
        last[ch] = r
        if r - l + 1 > best:
            best = r - l + 1
    return best
```

**参考实现** —— Python（[Java](./LC-003-longest-substring-without-repeating-characters/lc-3/sol.java) · [C++](./LC-003-longest-substring-without-repeating-characters/lc-3/sol.cpp) · [Go](./LC-003-longest-substring-without-repeating-characters/lc-3/sol.go)）

```python
# LC 3. Longest Substring Without Repeating Characters
from typing import Dict


def length_of_longest_substring(s: str) -> int:
    """LC 3: length of the longest substring of s with no repeating characters.

    One pass with a last-index map. For each character at position r, if it
    was seen at index `prev` AND `prev >= l` (i.e., the prior occurrence is
    inside the current window), jump l to prev + 1 in O(1) instead of
    shrinking step by step. The `prev >= l` guard is mandatory: stale
    entries from before the current window must be ignored, otherwise l
    can move backwards into discarded territory.
    O(n) time, O(min(n, sigma)) space; under ASCII sigma=128, space is O(1).
    """
    last_index: Dict[str, int] = {}
    l = 0
    best = 0
    for r, c in enumerate(s):
        if c in last_index and last_index[c] >= l:
            l = last_index[c] + 1
        last_index[c] = r
        if r - l + 1 > best:
            best = r - l + 1
    return best
```

> **Interactive widget:** [无重复字符的最长子串（LC 3）—— last-index 跳跃](../../../dsa/widgets/e-LC003-longest-substring.yml) —— 在网站上渲染；YAML 规范描述了各个关键帧。

*静态回放：在 `s = \"pwwkew\"` 上，窗口先扩到 `\"pw\"`（best=2）；`r=2` 处的第二个 `'w'` 触发跳跃，`l` 从 0 跳到 2；窗口在 `r=4` 长到 `\"wke\"`（best=3，即答案）；`r=5` 处的第二个重复字符 `'w'` 触发第二次跳跃，`l` 从 2 到 3，最后停在 `\"kew\"`，长度 3。六次迭代，两次跳跃，没有内层循环。*

组件在 `\"pwwkew\"` 上正好动画演示六次外层迭代。第 7 步是教学点：`last['p'] = 0` 此刻已经过期（它的值落在 `l = 3` 之下），后面若真的再来一个 `'p'`，守卫条件会默默忽略它。收缩循环版本没有这种风险，因为它根本没有跳跃；跳跃版本用一个收缩循环换来一个守卫条件，而那个守卫就是这笔交换的要害。

### last-index 跳跃：从 O(2n) 到均摊 O(n)

解法 2 和解法 3 的复杂度上界都是均摊 `O(n)`，差别在每个字符上的工作量。解法 2 里，每个字符通过 `r` 进窗口、通过 `l` 出窗口，均摊总量是 `2n` 次指针移动，而且这些移动是扎堆的（每次遇到重复都会带出一串 `l` 前进）。解法 3 里，每个字符只被外层循环读一次，内层工作至多是一次表读取、一次比较、一次赋值、一次表写入。总共 `n` 次外层迭代，每次内部 `O(1)`，没有扎堆。在对抗性输入上的墙钟基准显示，跳跃版本比收缩版本快 1.5 到 3 倍。[^3]

守卫条件 `last[ch] >= l` 就是区分「正确」和「悄悄答错」的那一行。处理 `\"pwwkew\"` 到下标 5 之后，`last['p'] = 0`，这条记录已经*过期*：`'p'` 早就不在窗口里了。如果后面再出现一个 `'p'`，无条件的跳跃 `l = last['p'] + 1 = 1` 会把 `l` 往*后*滑，滑进算法早已丢弃的领地。守卫条件的意思就是：只有上一次出现的下标落在*当前*窗口内时才跳。没有它，算法在任何字符于间隔之后重现的那一刻就会把窗口搞坏；有了它，过期记录会在下一次赋值时被默默覆盖掉。

两种等价写法：显式的 `if last[ch] >= l: l = last[ch] + 1`（更清楚），或者隐式的 `l = max(l, last.get(ch, -1) + 1)`（更紧凑）。两者都对。挑面试官读得更快的那个。[^4]

## 边界情况

**空串。** `\"\"` 返回 `0`。`for` 循环一次都不跑，`best` 停在初始值 0。C++ 和 Java 里惯用的 `s.size()` / `s.length()` 返回 0，循环干脆被跳过，不会有 off-by-one。

**单一重复字符。** `\"bbbbb\"` 返回 `1`。第一次之后的每次迭代都发现 `last['b']` 在窗口内，把 `l` 跳到 `prev + 1`（也就是跳到 `r`），更新 `last['b'] = r`，记下长度为 1 的窗口。窗口永远不会超过一个字符，`best` 也就不会超过 1。

**大小写混排。** LC 3 的字符集把 `'a'` 和 `'A'` 当成两个不同的 ASCII 码点。`\"AaAa\"` 返回 `2`，不是 1。算法与字符集无关，测试用例会覆盖这一点。

## 常见错误

**忘了 `last[ch] >= l` 这个守卫条件。** 最经典的一个。在 `\"abba\"` 上，处理到 `r = 2`（第二个 `'b'`）之后 `l = 2`。到 `r = 3` 时算法读到 `'a'`，发现 `last['a'] = 0`，无守卫的跳跃把 `l` 设成 1，*往后退*了。此时窗口覆盖 `s[1..3] = \"bba\"`，里面含重复的 `'b'`，返回的答案是 `1`，而正确答案是 `2`。修法是加上条件判断 `if last[ch] >= l`，或等价的 `max(l, ...)`。

**窗口长度差一。** 闭区间 `[l, r]` 的长度是 `r - l + 1`，不是 `r - l`。漏掉 `+ 1` 的提交会把每个窗口长度都报少 1：在 `\"abcabcbb\"` 上答案会变成 `2` 而不是 `3`。

**先更新 `best`、后修窗口（收缩循环写法）。** 如果把 `best = max(best, r - l + 1)` 写在内层 `while` *之前*，就会把仍然非法的窗口长度记进去，答案偏大。正确顺序是：扩展、修正、记录。这就是 algo.monster 陷阱清单里的 Pitfall 5，那里还附了一个错误顺序的实例。[^5]

**该用 `while` 却用了 `if`（收缩循环写法）。** 单步的 `if` 只从窗口头部去掉一个字符，重复却可能还在。多数输入能蒙过去，因为多数重复收缩一步就修好了；这个 bug 会在 `\"abba\"` 这类重复字符在好几格之前的输入上露头。解法 3 的跳跃写法绕开了这一整类 bug，这也是它值得上线的原因之一。[^5]

## 面试追问

**怎么返回真正的子串，而不只是长度？** 在 `best_len` 旁边再记一个 `best_l`。两者一起更新，但只在严格大于时更新 `best_l`，这样并列最长时*第一个*获胜；用 `>=` 更新则是*最后一个*获胜。返回 `s[best_l : best_l + best_len]`。[^4]

**如果约束换成「至多 K 个不同字符」而不是「无重复」呢？** K=1 就是「无重复」，也就是 LC 3，只是把重复判定换成 `len(window) <= K`。一般化写法：维护一张计数表和整数 `distinct`，某个键的计数从 `0 -> 1` 时加一，从 `1 -> 0` 时减一，在 `distinct > K` 时收缩。同样 `O(n)`，同样的骨架。K=2 就是 [LC 904 — Fruit Into Baskets](https://leetcode.com/problems/fruit-into-baskets/)；参数化的 LC 340 在付费墙后，是本章题目梯度里的 STRETCH 占位。

**对 ASCII 输入能完全不用哈希表吗？** 当字母表就是 LC 3 的约束集（可打印 ASCII，128 个码点）时，哈希表塌缩成一个初始化为 -1 的定长 `int[128]` 数组。上面的 C++ 和 Go 参考实现已经这么做了；Java 里是 `int[] lastIndex = new int[128]; Arrays.fill(lastIndex, -1)`，这样每次 put 和 get 都不必承受 `HashMap<Character, Integer>` 的自动装箱。在基准输入上大约快 3 倍，纯粹是因为哈希消失了。

**如果字符是一个一个到达的呢？** 算法本身就是一趟前向扫描、每个字符 `O(1)` 工作，所以离线和在线的版本是同一份代码。状态是 `(l, last_index, best)`；对外暴露 `next(c)`，更新这三者并吐出当前的 `best`。字母表无界的变体里 `last_index` 会长到 `O(n)`；在 LC 3 的 ASCII 约束下封顶 128。

从这道题通往它更难的表亲的桥是 [LC 76 — Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)，它保留可变窗口的脚手架，把重复判定换成覆盖集判定加一个整数计数器。循环形状一样，状态更难。

## 参考文献

[^1]: doocs/leetcode 题解仓库，"LC 3 Longest Substring Without Repeating Characters"。 https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0003.Longest%20Substring%20Without%20Repeating%20Characters/README_EN.md

[^2]: Cormen, Leiserson, Rivest, Stein，*Introduction to Algorithms*，第 4 版，MIT Press，2022，第 17 章（聚合分析）。

[^3]: LC 3 的 LeetCode 社区讨论帖（需登录，未镜像）。

[^4]: walkccc 题解，"LeetCode 3"。 https://walkccc.me/LeetCode/problems/3/

[^5]: AlgoMonster 题解，"3. Longest Substring Without Repeating Characters"。 https://algo.monster/liteproblems/3
