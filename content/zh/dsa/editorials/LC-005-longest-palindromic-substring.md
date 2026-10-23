---
lc_id: 5
title: "最长回文子串"
slug: LC-005-longest-palindromic-substring
primary_chapter: part-9-dynamic-programming/10-palindrome-dp
star: true
difficulty: medium
date_updated: 2026-05-25
languages: [python, java, cpp, go]
editorial_widget: e-LC005-longest-palindrome
---

# 最长回文子串

LC 5 最干净地说明了这件事：两个 Big-O 相同的算法，上线后的命运可以天差地别。二维 DP 表和中心扩展都是 O(n²)；但只有其中一个会在路上分配一百万个布尔值。在 LC 5 明写的 `n <= 1000` 上限下[^1]，暴力 O(n³) 要跑到约 10⁹ 次操作，必然超时；两个 O(n²) 解法只跑到约 10⁶ 次，远在一毫秒内结束。两个 `O(n²)` 之间怎么选，才是这一章的主题。

整节课的道理只有一句话。每个回文都有中心——要么是一个字符（奇数长度），要么是两个相邻字符之间的空隙（偶数长度）——所以枚举 `2n - 1` 个候选中心、在每个中心向外推，就够了。不用建表，不用分配，也没有按长度填表的顺序要记。

## 题目

给定字符串 `s`，返回 `s` 中任意一个最长的连续子串，正读反读相同。约束：`1 <= s.length <= 1000`，字符集限于数字与英文字母。若存在多个最长回文，判题器接受其中任意一个。[^1]

| 输入 | 期望输出 | 考察点 |
|---|---|---|
| `"babad"` | `"bab"` 或 `"aba"` | 多解策略；长度 3 的平局 |
| `"cbbd"` | `"bb"` | 偶数长度回文；只处理奇数中心的实现会漏掉 |
| `"a"` | `"a"` | 长度 1 的基线 |
| `"racecar"` | `"racecar"` | 本身就是回文；整串胜出 |

LC 5 要的是子串本身，不是它的长度。这个细节很小，但只跟踪 `max_len` 的候选人会为此白花五分钟。原文措辞见 [LC 5 Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)。

## 解法 1：暴力

把题面直译成代码。枚举所有 `i <= j` 的 `(i, j)`，用 O(j - i) 检查子串是否为回文，并记录最长的那次命中。

```python
# 暴力：O(n^3) —— 这就是我们要替换掉的东西
def longest_palindrome_brute(s: str) -> str:
    n = len(s)
    best_l, best_r = 0, 0

    def is_palin(lo: int, hi: int) -> bool:
        while lo < hi:
            if s[lo] != s[hi]:
                return False
            lo += 1
            hi -= 1
        return True

    for i in range(n):
        for j in range(i, n):
            if (j - i) > (best_r - best_l) and is_palin(i, j):
                best_l, best_r = i, j
    return s[best_l:best_r + 1]
```

被浪费的工作在倒数第二行上就能看见：`is_palin(i, j)` 先检查 `s[i] == s[j]`，然后从头把 `s[i+1..j-1]` 的整段回文判定再跑一遍。而那段内层区间本身也是一个子串，算法会在另一个 `(i, j)` 上重新计算它的回文性。时间 `O(n³)`，空间 `O(1)`。`n = 1000` 时最坏情况约 `5 × 10⁸` 次比较，在 LeetCode 给 Python 和 Java 的约 1 秒预算下会 TLE。[^2]

修法是别再重算 `is_palin(i+1, j-1)`。要么把它物化到表里、只读一次；要么把回文判定折进迭代里，让这份工作被隐式复用。两种修法都能到 O(n²)；区别在空间，以及迭代长什么样。

## 解法 2：二维 DP 表

定义 `is_palin[i][j] = True` 当且仅当 `s[i..j]` 是回文。递推式读作：

```text
is_palin[i][j] = (s[i] == s[j]) AND (j - i < 2 OR is_palin[i+1][j-1])
```

长度 1 的格子平凡地为 `True`；长度 2 的格子当且仅当两个字符相同才为 `True`。对长度 `L >= 3`，该格子依赖严格更内层、长度为 `L - 2` 的区间，也就是*下面*那一行、*左边*那一列。[^3]

关键在于填表顺序按长度走，而不是按行走。格子 `is_palin[i][j]` 要读 `is_palin[i+1][j-1]`。自上而下填（`for i in 0..n: for j in i..n`）会读到还没算出来的格子；这个 bug 的表象是：长度 1、长度 2 的对角线正确，接着所有长度 3 以上的回文都报成不存在。正确的外层循环按长度 `L = 1, 2, 3, ..., n` 迭代，再按起点 `i` 迭代，`j = i + L - 1`。等你算到长度 L 的格子时，所有长度 (L-2) 的格子都已经有值了。

时间 `O(n²)`，空间 `O(n²)`。`n = 1000` 时约 `10⁶` 次操作、约 `10⁶` 个布尔值（约 1 MB）。能过。表版本同时也是 LC 647（数回文子串，同一个谓词）和 LC 132（最少分割，在它上面再叠一层一维 DP）的天然铺垫。下一个解法渐近时间相同，且零分配。

## 解法 3：中心扩展（最优）

每个回文都有中心。奇数长度回文的中心是一个字符；偶数长度回文的中心是两个相邻字符之间的空隙。候选中心恰好有 `2n - 1` 个（`n` 个字符中心加 `n - 1` 个字符之间中心）。对每个候选，只要 `s[left] == s[right]` 就把两个指针向外推。所有中心里扩展得最长的那次就是答案。[^4][^2]

```python
def longest_palindrome(s: str) -> str:
    if not s:
        return ""

    def expand(left: int, right: int) -> tuple[int, int]:
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return left + 1, right - 1

    best_l, best_r = 0, 0
    for i in range(len(s)):
        l1, r1 = expand(i, i)         # 奇数长度中心在 i
        l2, r2 = expand(i, i + 1)     # 偶数长度中心在 i 与 i+1 之间
        if r1 - l1 > best_r - best_l:
            best_l, best_r = l1, r1
        if r2 - l2 > best_r - best_l:
            best_l, best_r = l2, r2
    return s[best_l:best_r + 1]
```

**解法** — Python（[Java](./LC-005-longest-palindromic-substring/lc-5/sol.java) · [C++](./LC-005-longest-palindromic-substring/lc-5/sol.cpp) · [Go](./LC-005-longest-palindromic-substring/lc-5/sol.go)）

```python
# LC 5. Longest Palindromic Substring
from typing import Tuple


def longest_palindrome(s: str) -> str:
    """LC 5: longest contiguous palindromic substring of s.

    Expand around each of the 2n - 1 candidate centers (n character-centers
    plus n - 1 between-character centers). For each center, push outward
    while s[left] == s[right]; the loop terminates one step past the last
    valid match, so the maximal palindrome is s[left+1 : right] when the
    helper returns the corrected (left+1, right-1) pair.
    O(n^2) time, O(1) extra space.
    """
    if not s:
        return ""

    def expand(left: int, right: int) -> Tuple[int, int]:
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return left + 1, right - 1

    best_l, best_r = 0, 0
    for i in range(len(s)):
        l1, r1 = expand(i, i)            # 奇数长度中心在 i
        l2, r2 = expand(i, i + 1)        # 偶数长度中心在 i 与 i+1 之间
        if r1 - l1 > best_r - best_l:
            best_l, best_r = l1, r1
        if r2 - l2 > best_r - best_l:
            best_l, best_r = l2, r2
    return s[best_l:best_r + 1]
```

> **Interactive widget:** [最长回文子串（LC 5）—— 中心扩展](../../../dsa/widgets/e-LC005-longest-palindrome.yml) — 在网站上渲染；YAML 规范描述了各个关键帧。

*静态回退：`s = "babad"` 时，算法走过 9 个中心（5 个奇数、4 个偶数）。i=1 时奇数扩展找到 `"bab"`，把当前最优更新为长度 3。i=2 时奇数扩展找到 `"aba"`，也是长度 3；严格的 `>` 拒绝平局，所以答案仍是 `"bab"`。其余中心找不到更长的回文。*

`expand` 这个辅助函数是承重件。它的循环在指针不一致或越出字符串时终止，也就是说*最后一次有效匹配*发生在上一轮迭代。返回 `(left + 1, right - 1)` 就是在修正这一步的过冲。`expand(i, i)` 与 `expand(i, i + 1)` 这对调用，正是用来同时处理奇数长度和偶数长度中心的手段，不必像 Manacher 算法和教科书里那个「Slower algorithm」一样先做输入倍增[^4]。

时间 `O(n²)`：`2n - 1` 个中心各最多扩展 `n/2` 步，所有中心的总工作量按等差级数论证不超过 `n²/2`。[^4] 空间 `O(1)` 额外。组件演示 `"babad"` 这一条轨迹，是因为只有这个输入能暴露严格更新与不严格更新的守卫差别——i=2 的平局在更短的输入上就不见了。

### 实践上为什么胜过 DP 表

渐近时间相同。零分配。没有按长度填表的顺序要背。常数更小：每次迭代最多碰两个字符并比较它们；填表的迭代要碰三个格子（`is_palin[i+1][j-1]`、`s[i]`、`s[j]`）再加一次写入。扩展版一直待在 cache 里，表版不是。

Manacher 算法靠回文的镜像性质做到 O(n)：当一个以 `c` 为中心的长回文已经延伸到右边界 `r` 时，任何位置 `i < r` 都能免费继承至少 `min(r - i, p[mirror_of_i])` 的半径，用「中心不后退、右边界只前进」的势能论证把总工作量摊还到线性。[^5] `n = 1000` 时这是杀鸡用牛刀；`10⁶` 对 `10³`，两边都是亚毫秒。要记住它名字的理由：`n >= 10⁵` 时，中心扩展在全同字符的最坏输入上会 TLE，Manacher 就从可选变成了承重。完整代码大约是中心法的四倍长，35 分钟的面试里很少有人手写。说出算法名字、画出镜像直觉的草图，就是期望的回答。

## 边界情况

**空串。** LC 的约束保证 `s.length >= 1`，但生产级安全的代码会在 `n == 0` 时短路，避免读 `s[0]` 越界。Python 参考实现立刻返回 `""`；另外四份实现也一样。

**偶数长度回文（`"abba"`、`"cbbd"`）。** 只处理奇数中心的实现最经典的坑。对任意 `i`，`expand(i, i)` 在 `"cbbd"` 上最多找到长度 1 的回文（每个字符都是它自己的回文）；`expand(1, 2)` 找到长度 2 的 `"bb"`。少了第二次调用，算法会在有偶数长度答案的输入上返回长度 1 的子串。

**全同字符串（`"aaaa"`）。** 每个区间都是回文，所以答案是整串。中间下标的奇数中心会一路走到两端边界；这个输入恰好就是算法时间上的最坏情况，每个中心扩展到半串，总工作量 `n²/2` 正好吻合上界。[^4]

## 常见错误

**只跑奇数长度中心。** 对每个 `i` 调用 `expand(s, i, i)` 而跳过 `expand(s, i, i + 1)`，在 `"cbbd"` 上会返回 `'c'`、`'b'`、`'d'` 其中之一，在 `"abba"` 上返回 `'a'`。思路上的修法是换框架：偶数长度的回文锚定在两个字符*之间*，不在某一个字符上。两种锚点都必需。

**返回 `(left, right)` 而不是 `(left + 1, right - 1)`。** 扩展循环的终止条件是 `s[left] != s[right]` 或越界。循环结束后，`left` 和 `right` 已经指到最后一个有效匹配的后一步。直接返回 `(left, right)` 会把不匹配的字符也含进来；在 `"cbbd"` 上，`i = 1` 的奇数扩展会报出长度 4（整串）而不是长度 1。在 `expand` 内部把修正后的窗口算一次，算法其余部分就能干净地消费它。

**返回长度而不是子串。** 迭代时只跟踪 `max_len`，回答的是另一个问题。LC 5 要子串；把 `(start, max_len)` 一起跟踪（或像参考实现那样用 `(best_l, best_r)`），最后切一次。CLRS 习题 14-2 里那个最长回文*子序列*的教科书 DP 递推，格子里存的是标量长度[^3]，它训出的是对 LC 5 错误的条件反射；见证元必须一起跟踪。

**在扩展之后按 `r - l - 1` 算长度。** 调试 off-by-one 时，候选人常常先改 `expand` 的返回值，然后在下游某处写 `length = r - l - 1`，想「把过冲减回去」。两处修正叠加就成了新的 bug。只在一个地方做这套算术（`expand` 内部），让其余代码在真实的 `[bestL, bestR]` 坐标下工作。

## 面试追问

**如果要数出所有回文子串，而不是返回最长的那一个呢？** 那是 [LC 647](https://leetcode.com/problems/palindromic-substrings/)。同一个中心扩展模板。对每个中心，每次匹配成功就把计数器加一，而不是跟踪最长。`O(n²)` 时间，`O(1)` 空间，在参考实现上改两行。Amazon 很爱问这道。[^4]

**最长回文*子序列*呢？** 名字撞了，但这是另一道题：[LC 516](https://leetcode.com/problems/longest-palindromic-subsequence/)。子串（LC 5）要求两端匹配，并且内层区间是回文——严格的 AND，一旦某一环断了「否」就沿链传下去。子序列在两个端点不匹配时放宽成 `max(dp[i+1][j], dp[i][j-1])`，因为允许跳过一个端点。在 `"bbbab"` 上，LC 5 返回 `"bbb"`（长度 3，连续）；LC 516 返回 4（`"bbbb"`，跳过那个 `'a'`）。LC 516 是 CLRS 习题 14-2 / 15-2 的经典设定[^3]，而且它不会屈服于中心扩展。把这两者分清楚，正是这道追问的全部意义。

**你会怎么把字符串分割成若干回文？** 那是 [LC 131](https://leetcode.com/problems/palindrome-partitioning/)，在回文判定之上做回溯。先用解法 2 的 `is_palin[i][j]` 表在 O(n²) 内预处理，然后从每个位置递归到所有「前缀是回文」的更后位置，枚举分割方案。贵的部分是分割枚举（最坏情况指数级），不是回文谓词；这张表的价值就体现在这里。

**如果字符是一个一个到达的呢？** 这就是 Manacher 最初的框架。当算法要在每个前缀上维护「目前为止的最长回文」时，中心扩展不能干净地延展：每个新字符都可能让靠近右边界的那些中心已经做完的工作失效。Manacher 1975 年论文的标题是「A new linear-time *on-line* algorithm for finding the smallest initial palindrome of a string」[^5]；*on-line* 这个限定词就是整个面试答案。右边界 `r` 与中心 `c` 随着每个字符推进，摊还代价 O(1)。

## 参考文献

[^1]: LeetCode, "5. Longest Palindromic Substring." https://leetcode.com/problems/longest-palindromic-substring/

[^2]: walkccc, "LeetCode 5." https://walkccc.me/LeetCode/problems/5/

[^3]: walkccc, "CLRS Problem 15-2: Longest palindrome subsequence." https://walkccc.me/CLRS/Chap15/Problems/15-2/

[^4]: Wikipedia, "Longest palindromic substring." https://en.wikipedia.org/wiki/Longest_palindromic_substring

[^5]: Manacher, "A new linear-time on-line algorithm for finding the smallest initial palindrome of a string," JACM 22(3), 1975. https://doi.org/10.1145/321892.321896
