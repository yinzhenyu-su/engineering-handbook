---
lc_id: 11
title: "盛最多水的容器"
slug: LC-011-container-with-most-water
primary_chapter: part-3-pointers-window-prefix/00-two-pointers-opposite
star: true
difficulty: medium
date_updated: 2026-05-25
languages: [python, java, cpp, go]
editorial_widget: e-LC011-container
---

# 盛最多水的容器

LC 11 是双指针家族里最干净的一道案例分析证明题。算法只有六行。真正被考的是你必须当面讲清楚的那句话：*移动较高的指针不可能带来任何好处*。这一句就是整道题；剩下的全是实现。

## 题目

给定长度为 `n` 的整数数组 `height`，把每个元素看作立在 x 轴下标 `i` 处的一面竖直墙。任取两个下标 `i < j`。这两面墙之间盛的水受*较矮*那面墙限制，并且横向铺满两者之间的整段距离，所以面积是 `min(height[i], height[j]) * (j - i)`。返回所有下标对中最大的面积。约束：`2 <= n <= 10⁵`，`0 <= height[i] <= 10⁴`。[^1]

| 输入 | 期望输出 | 考察点 |
|---|---|---|
| `[1, 8, 6, 2, 5, 4, 8, 3, 7]` | `49` | 经典用例；最优对是 `(1, 8)`，面积为 `min(8, 7) * 7` |
| `[1, 1]` | `1` | `n` 取最小值；宽度是 `1` 而不是 `2` |
| `[4, 3, 2, 1, 4]` | `16` | 两端墙同高；两者之间的间距就是答案 |

原文措辞见 [LC 11 Container With Most Water](https://leetcode.com/problems/container-with-most-water/)。

## 解法 1：暴力

把题面直接翻译成代码。枚举每一个无序对 `i < j`，算出面积，保留最大值。

```python
# 暴力：O(n²) —— 这就是我们要替换掉的东西
def max_area_brute(height):
    n = len(height)
    best = 0
    for i in range(n):
        for j in range(i + 1, n):
            area = min(height[i], height[j]) * (j - i)
            if area > best:
                best = area
    return best
```

在所有输入上都正确，在判题器在乎的那些输入上必死。`n = 10⁵` 时内层循环体大约要跑 `n * (n - 1) / 2 ≈ 5 × 10⁹` 次。LC 单个测试点的时间预算大约一秒，在 Python 里大约能换来 `10⁸` 次简单操作，C++ 能再多几倍。`n` 过了 `10⁴` 左右就超时。在写优化之前先说一句*“这是 `O(n²)`；约束想要的是 `O(n)`”*，这就是面试要的信号。

## 解法 2：双指针（最优）

两个指针从数组两端开始。算出候选面积。把位于严格较矮那面墙上的指针往里挪一格；两墙等高时任选一侧（四份参考实现采用的惯例是走 `else` 分支，也就是移动右指针）。重复到两个指针交错为止。

```python
def max_area(height):
    l, r = 0, len(height) - 1
    best = 0
    while l < r:
        h = min(height[l], height[r])
        if h * (r - l) > best:
            best = h * (r - l)
        if height[l] < height[r]:
            l += 1
        else:
            r -= 1
    return best
```

**解法** — Python（[Java](./LC-011-container-with-most-water/lc-11/sol.java) · [C++](./LC-011-container-with-most-water/lc-11/sol.cpp) · [Go](./LC-011-container-with-most-water/lc-11/sol.go)）

```python
# LC 11. Container With Most Water
from typing import List


def max_area(height: List[int]) -> int:
    """LC 11: maximum water held between two walls in `height`.

    Two pointers, opposite ends. At each step, compute the candidate area
    `min(h[l], h[r]) * (r - l)`, then advance the pointer at the strictly
    shorter wall (ties: advance the right pointer, by convention). The
    case-analysis proof is in the editorial: advancing the taller wall
    cannot improve the answer because the shorter wall still binds and
    the width strictly shrinks. O(n) time, O(1) space.
    """
    left, right = 0, len(height) - 1
    best = 0
    while left < right:
        h_l, h_r = height[left], height[right]
        width = right - left
        if h_l < h_r:
            area = h_l * width
            if area > best:
                best = area
            left += 1
        else:
            area = h_r * width
            if area > best:
                best = area
            right -= 1
    return best
```

> **Interactive widget:** [Container With Most Water (LC 11) —— 贪心双指针收缩](../../../dsa/widgets/e-LC011-container.yml) —— 在网站上渲染；YAML 规范描述了各个关键帧。

*静态回退：在 `[1, 8, 6, 2, 5, 4, 8, 3, 7]` 上，第 1 步把两个指针放在下标 0 和 8（高度 1 和 7），算出面积 8。第 2 步把 `l` 挪到 1（高度 8），算出 `min(8, 7) * 7 = 49`，即最优解。剩下六步里 `r` 一路左移，始终没能超过 49；指针交错时循环结束。*

循环最多跑 `n - 1` 轮，因为每步只把一个指针挪一格。时间 `O(n)`，空间 `O(1)`。`n = 10⁵` 时一位数毫秒就跑完。[^2]

### 为什么移动较高的指针永远没有用

案例分析本身很短，但步骤的顺序很关键。把它写清楚一次，你以后就不用再写了。

设两个指针在 `(l, r)`，且 `height[l] < height[r]`。候选面积是 `min(height[l], height[r]) * (r - l) = height[l] * (r - l)`，因为左墙较矮，高度由它决定。

可选的动作有两个：要么把 `r` 往里挪一格（朝下标更小的方向），要么把 `l` 往里挪一格（朝下标更大的方向）。看这两种动作分别把新一对下标的候选面积变成什么。

**把 `r` 往里挪，`l` 不动。** 新的一对是 `(l, r')`，其中 `r' < r`。新宽度是 `r' - l`，严格小于 `r - l`。新的高度上限是 `min(height[l], height[r'])`。右墙的新高度有两种情况：

- `height[r'] >= height[l]`。上限仍停在 `height[l]`（还是左墙决定高度）。新面积是 `height[l] * (r' - l) < height[l] * (r - l)`，严格更差。
- `height[r'] < height[l]`。上限降到 `height[r']`，严格小于 `height[l]`。新面积是 `height[r'] * (r' - l)`。上限变小了，宽度也变小了，新面积严格小于 `height[l] * (r - l)`，同样严格更差。

两种情况里，不管 `height[r']` 最后是多少，面积都只会变小。固定 `l` 让 `r` 一路往里走，经过的每一对都严格差于当前这一对，没有一对能超过手上的答案。

**把 `l` 往里挪，`r` 不动。** 新的一对是 `(l', r)`，其中 `l' > l`。新宽度是 `r - l'`，严格更小。新的高度上限是 `min(height[l'], height[r])`。若 `height[l'] <= height[l]`，上限至多是 `height[l]`，面积至多 `height[l] * (r - l')`，严格小于当前值。但若 `height[l'] > height[l]`，高度上限就*抬高*了：此时它由 `height[l']` 和 `height[r]` 中较小者决定，而这两个都大于 `height[l]`。面积会不会变大，取决于高度的增益能否抵掉宽度的损失。关键在于它*有可能*变大。挪 `l` 是唯一具备这个性质的动作。

所以在任何一个状态下，挪动较矮那一侧的指针都是唯一可能让答案变好的方向。挪较高的指针是被支配的：刚被放弃的那一对不比手上的差，而那一侧的后续每一对也不会更好。我们可以直接把所有 `r' < r` 的 `(l, r')` 全部丢掉、一个都不检查，仍然能找到最优解，因为它们都不可能比 `(l, r)` 盛更多水。当 `height[l] > height[r]` 时，对称的论证同样成立。

关键词是*被支配*（dominated）。这个证明并没有说挪较矮那面墙就能找到最优解；它说的是挪较高那面墙*不可能*找到比手上更好的东西。算法靠排除法推进：每一步丢掉搜索空间里的一条射线（被放弃的指针那一侧的所有下标对），因为证明保证了最优解不在里面。

这与 CLRS 第 16 章里让活动选择贪心成立的交换论证是同一个形状：每一步证明局部的贪心动作不被任何别的动作支配，全局最优就由归纳法跟上。[^3]

## 边界情况

**`n = 2` 的最小规模。** 像 `[1, 1]` 或 `[5, 3]` 这样的输入只有一对要评估。宽度是 `j - i = 1`，不是 `2`；两根柱子在下标 0 和 1，它们之间的间距就是一个单位。`[1, 1]` 返回 `1`；`[5, 3]` 返回 `3`。宽度差一位的错误在这个用例上会立刻被判题器抓住。

**两墙等高。** 像 `[5, 5, 5]` 这样的输入，第一次比较时没有严格较矮的墙。算法走 `else` 分支，挪动 `r`（四份参考实现都用的严格 `<`，所以相等会落到移动右指针）。挪哪一侧都是对的；证明里的对称论证覆盖了同高的情况，前提是“移动较矮的一侧，同高时把任一侧视作较矮”。[^4] 两个方向都能得到同一个答案。

**高度严格递减。** 在 `[5, 4, 3, 2, 1]` 上右墙永远更矮，所以每一轮都挪 `r`。第一次探测在 `(0, 4)`，得到 `min(5, 1) * 4 = 4`；最优落在 `(0, 3)`，`min(5, 2) * 3 = 6`。单调数组退化成单向扫描。

## 常见错误

**把宽度写成 `r - l + 1`。** LC 11 最常见的错。墙立在整数 x 坐标上，厚度为零；两墙之间盛的水是几何上的*间距*，不是它们框住的格子数。在 `[1, 1]` 上这个错公式返回 `2`，判题器在第一个隐藏用例上就拒。修法：宽度永远是 `r - l`。

**两墙等高时把两个指针一起挪。** 标准算法每轮只挪一个指针，同高时挪右指针。同高时一起挪，在你丢掉的那一侧正好离同高状态只差一步就是最优的输入上会错。在 `[5, 1, 5]` 上，初始对 `(0, 2)` 高度同为 5，面积 `5 * 2 = 10`；一起挪会落到 `l = r = 1`，循环直接退出，`(0, 1)` 和 `(1, 2)` 两对都没检查。只挪一个能保住搜索，返回的仍是 `10`。单侧移动哪种都行；两侧一起挪不行。

**挪较高的那个指针。** 跳过证明、直接试对称写法（挪较高的墙）的读者，写出来的代码会在第一个非对称用例上挂掉。在 `[1, 8, 6, 2, 5, 4, 8, 3, 7]` 上，第 1 步挪较高的墙意味着 `r` 从下标 8 移到下标 7（高度 3），把属于最优解的那面高度 7 的墙放弃了。算法返回 16，不是 49。这个证明存在的意义正是防止这个错；读一遍证明比调这个 bug 便宜。

**只在严格变好时更新 `best`。** 写 `if area > best` 是对的（题目求最大值，等于当前值时不更新等于没更新）；写 `if area >= best` 也对，只是多余。要防的是反面：干脆不比，直接 `best = area` 无条件赋值。只要后面某一轮的面积小于 `best`，这个无条件赋值就会把正确答案覆盖成垃圾。

## 面试追问

**那接雨水（Trapping Rain Water）呢？** 自然的 Hard 级追问是 [LC 42](https://leetcode.com/problems/trapping-rain-water/)，看起来是同一道题，其实不是。LC 42 累加的是*每个格子上方*能存的水，不是两面特定墙之间盛的水。双指针的骨架能搬过去——指针在两端，挪较矮的一侧——但逐格公式是 `min(left_max, right_max) - height[i]`，滑动的左右最大值要随指针内移维护。同一个家族，记账更难；LC 11 的证明就是 LC 42 证明的热身。[^2]

**如果要返回下标而不只是面积呢？** 在 `best` 旁边再维护 `best_l` 和 `best_r`，每次严格变好时一起更新。算法结构不变，只是记账变多。多数面试官在你写完面积版之后会问这个版本，用来确认你清楚自己找到的是哪一对。

**如果柱子是流式到达的呢？** 双指针需要两端都能随机访问；从左到右的流在第二根柱子到达时没法放置右指针。流式版本是另一道题，要维护一个“仍可能与未来右墙配对的候选左墙”的单调结构，通常用单调栈解决。这里的面试信号是：能认出随机访问是双指针家族的前提条件，而不是无关细节。

## 参考文献

[^1]: doocs/leetcode, "LC 11 Container With Most Water." https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0011.Container%20With%20Most%20Water/README_EN.md

[^2]: NeetCode, "Container With Most Water." https://neetcode.io/problems/max-water-container

[^3]: Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press, 2022, Ch. 16 (greedy algorithms).

[^4]: scanny et al., "Proof for LeetCode 11," CS Stack Exchange, 2020. https://cs.stackexchange.com/questions/121472/proof-for-leetcode-11-container-with-most-water-problem
