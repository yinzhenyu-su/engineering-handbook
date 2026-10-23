---
lc_id: 15
title: "三数之和"
slug: LC-015-3sum
primary_chapter: part-3-pointers-window-prefix/00-two-pointers-opposite
star: true
difficulty: medium
date_updated: 2026-05-25
languages: [python, java, cpp, go]
editorial_widget: e-LC015-3sum
---

# 三数之和

三数之和这一章，是大多数候选人第一次把去重当成真正的算法问题来对待的地方。算术部分很简单：固定一个元素，剩下的就是 2Sum。难的是既不借助哈希集合，也不做「排序后的元组」这种规范化，还不拖慢渐进上界，却把每一个不同的三元组恰好报一次。整章的分量都压在三行指针跳跃的纪律上。下面这篇题解，讲的几乎全是这三行。

归约只有一句话。把数组排好序，固定一个锚点 `nums[i]`，内层子问题就变成「找出所有满足 `b + c = -nums[i]` 的数对 `(b, c)`」——也就是有序数组上的 [Two Sum](./LC-001-two-sum.md)。`O(n^2)` 的总开销在比较模型下已经渐进最优，所以一旦归约到双指针，剩下的工作是正确性，不是速度。

## 题目

给定整数数组 `nums`，返回所有和为 0、且由三个不同位置上的值组成的不重复三元组。两个三元组只要所含值的多重集相同（不计顺序）就算相等；答案集中每个不同的三元组只能出现一次。约束：`3 <= nums.length <= 3000`，`-10^5 <= nums[i] <= 10^5`。难度中等。[^1]

| 输入 | 期望输出 | 考察点 |
|---|---|---|
| `[-1, 0, 1, 2, -1, -4]` | `[[-1, -1, 2], [-1, 0, 1]]` | 经典用例；两个有效三元组，共用一个锚点值 |
| `[0, 0, 0]` | `[[0, 0, 0]]` | 三个相同的值只产出一个三元组，不是三个 |
| `[0, 0, 0, 0]` | `[[0, 0, 0]]` | 四个相同的值仍然只产出一个三元组 |
| `[1, 2, 3]` | `[]` | 全正输入；没有任何三元组之和为 0 |

输出是**值**的列表，不是下标。正是这个区别让三数之和比 [Two Sum](./LC-001-two-sum.md) 难：以索引为键的哈希表并不会天然压掉「值等价」的重复三元组。三元组必须唯一这条规则是承重约束，下面每个解法都按它把唯一性落实得干不干净来评分。原文措辞见 [LC 15 3Sum](https://leetcode.com/problems/3sum/)。

## 解法 1：暴力

枚举三个下标 `i < j < k`，检查和，命中的结果先排序再存进集合里完成去重。

```python
# 暴力：O(n^3) 时间，O(n^2) 空间 —— 这就是我们要替换掉的东西
def three_sum_brute(nums):
    seen = set()
    out = []
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            for k in range(j + 1, n):
                if nums[i] + nums[j] + nums[k] == 0:
                    triplet = tuple(sorted((nums[i], nums[j], nums[k])))
                    if triplet not in seen:
                        seen.add(triplet)
                        out.append(list(triplet))
    return out
```

`tuple(sorted(...))` 这步规范化是强制的。从下标 `(0, 3, 4)` 和 `(0, 4, 3)` 取到的两个三元组是同一个多重集，却是两个不同的有序元组；少了这层内部排序，两个哈希键就不一样，重复项会漏出去。

时间 `O(n^3)`。集合最坏占 `O(n^2)` 空间。`n = 3000` 时暴力解大约要做 `2.7 * 10^10` 次求和检查，在 LC 的判题机上超过 `n = 500` 左右就开始超时。面试里该做的动作照样是把它写出来。它给讨论定了锚，让你把复杂度念出声，也换来「现在我来加速」这个转折——后面整段对话都需要它。

## 解法 2：每个锚点配哈希集合

这是第二直觉，也是很多人用哈希表做完 [Two Sum](./LC-001-two-sum.md) 之后会伸手去拿的解法。先排序，然后对每个锚点 `i`，在后缀上用哈希集合跑一次目标为 `-nums[i]` 的 Two Sum。

```python
# O(n^2) 时间，O(n) 空间 —— 正确，但边界处理很别扭
def three_sum_hash(nums):
    nums = sorted(nums)
    n = len(nums)
    out = []
    for i in range(n - 2):
        if nums[i] > 0:
            break
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        seen = set()
        j = i + 1
        while j < n:
            complement = -nums[i] - nums[j]
            if complement in seen:
                out.append([nums[i], complement, nums[j]])
                while j + 1 < n and nums[j] == nums[j + 1]:
                    j += 1
            seen.add(nums[j])
            j += 1
    return out
```

「三数之和等于锚点加两数之和」这个归约是承重洞察，要说出来。哈希版本为它付了两次代价：每个锚点都要多花 `O(n)` 空间养那张集合，而且在常数因子上比排序加指针版本里的数组索引更慢。值得写一遍确认归约成立；不值得交上去。

## 解法 3：排序加双指针（最优）

先排一次序。对每个锚点 `i`，把指针放在 `l = i + 1` 和 `r = n - 1`，向中间收拢，寻找 `nums[l] + nums[r] == -nums[i]`。每次命中之后，两个指针都前进，并跳过刚输出的值在两侧的重复项。外层循环在 `nums[i] > 0` 时提前退出，因为在有序数组上，以正数为锚点的三元组之和不可能为零。

```python
def three_sum(nums):
    nums = sorted(nums)
    n = len(nums)
    out = []
    for i in range(n - 2):
        if nums[i] > 0:                              # 提前退出
            break
        if i > 0 and nums[i] == nums[i - 1]:         # (a) 跳过重复锚点
            continue
        target = -nums[i]
        l, r = i + 1, n - 1
        while l < r:
            s = nums[l] + nums[r]
            if s < target:
                l += 1
            elif s > target:
                r -= 1
            else:
                out.append([nums[i], nums[l], nums[r]])
                l += 1
                r -= 1
                while l < r and nums[l] == nums[l - 1]:    # (b) 跳过左侧重复值
                    l += 1
                while l < r and nums[r] == nums[r + 1]:    # (c) 跳过右侧重复值
                    r -= 1
    return out
```

**解法** — Python（[Java](./LC-015-3sum/lc-15/sol.java) · [C++](./LC-015-3sum/lc-15/sol.cpp) · [Go](./LC-015-3sum/lc-15/sol.go)）

```python
# LC 15. 3Sum
from typing import List


def three_sum(nums: List[int]) -> List[List[int]]:
    """LC 15: return every unique triplet of values summing to zero.

    Sort, then for each anchor i run a converging two-pointer sweep over the
    suffix targeting -nums[i]. Three duplicate-skip lines do all the work:
    skip equal anchors, and after each hit advance both pointers past their
    own value runs. O(n^2) time, O(1) auxiliary space (sort excluded).
    """
    nums = sorted(nums)
    n = len(nums)
    out: List[List[int]] = []
    for i in range(n - 2):
        if nums[i] > 0:
            break
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        target = -nums[i]
        l, r = i + 1, n - 1
        while l < r:
            s = nums[l] + nums[r]
            if s < target:
                l += 1
            elif s > target:
                r -= 1
            else:
                out.append([nums[i], nums[l], nums[r]])
                l += 1
                r -= 1
                while l < r and nums[l] == nums[l - 1]:
                    l += 1
                while l < r and nums[r] == nums[r + 1]:
                    r -= 1
    return out
```

> **Interactive widget:** [三数之和（LC 15）—— 排序 + 锚点 + 双端对撞，带重复跳过](../../../dsa/widgets/e-LC015-3sum.yml) — 在网站上渲染；YAML 规范描述了各个关键帧。

*静态回退：在 `nums = [-1, 0, 1, 2, -1, -4]` 上，排序后得到 `[-4, -1, -1, 0, 1, 2]`。锚点 `-4` 扫过整个后缀，没有命中。下标 1 处的锚点 `-1` 先找到 `(-1, 2)`，再找到 `(0, 1)`，两个三元组都输出。下标 2 处的锚点 `-1` 被规则 (a) 跳过。锚点 `0` 及之后的都不贡献结果。最终结果：`[[-1, -1, 2], [-1, 0, 1]]`。*

时间 `O(n^2)`：外层是 `O(n)` 个锚点，每次内层扫描是 `O(n)`，因为 `r - l` 每轮缩小 1。空间为 `O(1)` 辅助（排序的 `O(log n)` 栈相对外层循环的工作量可以忽略，按惯例不计）。这个上界与 3SUM 的线性决策树下界相符，所以在比较模型下双指针已经渐进最优。[^2]

### 不用集合的去重

这一节才是整篇题解存在的理由。双指针的骨架很直白；三条跳跃规则不是。漏掉任何一条，算法都会在你根本想不到要去测的输入上，悄无声息地输出重复三元组。

跳跃点一共只有三个，顺序如下：

**（a）选定锚点 `i` 之后。** 若 `i > 0 and nums[i] == nums[i-1]`，整个内层扫描直接跳过。上一个取到这个值的锚点，已经把包含它的每一个三元组都枚举过了；再跑一遍只会换个起始下标把同样的三元组重新输出一次。在 `[-1, -1, 0, 1]` 上，下标 0 处的锚点 `-1` 找到 `(0, 1)`，输出 `[-1, 0, 1]`。少了这次跳跃，下标 1 处的锚点 `-1` 也会找到 `(0, 1)`，把 `[-1, 0, 1]` 再输出一遍。

**（b）命中之后，把 `l` 推过它所在的同值段。** 三元组一旦输出，`l += 1` 就移到下一个候选。若这个候选与刚用过的位置同值（`nums[l] == nums[l-1]`），同一个三元组就会带着 `r` 侧另一个伙伴再输出一次。`while` 循环把 `l` 一直往前推，直到值发生变化。

**（c）命中之后，把 `r` 推过它所在的同值段。** 与（b）对称。右侧是同一个问题的镜像：一段截至刚用过的位置的同值区间，会和下一个 `l` 重新配对，输出同一个三元组。

三种情况下的不变式是同一个：每次守卫触发之后，活跃指针所在的值，与当前锚点下已输出三元组里用过的任何值都不相同。能同时暴露这三种失效模式的用例是 `[-2, -2, 0, 0, 2, 2]`：下标 0 处的锚点 `-2` 在 `(l=2, r=4)` 找到 `(0, 2)`，输出 `[-2, 0, 2]`。少了（b），下一轮 `l=3` 会找到同一个三元组，因为 `r` 滑了一位，落在重复的 `2` 上。少了（c），对称的失效发生在另一侧。少了（a），第二个 `-2` 锚点会把一切都重新枚举一遍。三处跳跃加起来七行代码，正是它把算法留在 `O(n²)` 时间和 `O(1)` 额外空间，而不是去交那笔 `O(n²)` 的集合代价。[^3]

有个更诱人的替代做法：记下上一次输出三元组的各个分量（`preva, prevb, prevc`），任何新命中都先拒掉。在 `[-1, 0, 1, 2, -1, -4]` 上，算法输出 `[-1, -1, 2]`，把 `preva` 设为 `-1`，接着就因为新锚点的值也等于 `preva` 而拒绝输出 `[-1, 0, 1]`。修法是丢掉按值追踪，改用上面按位置跳跃的做法。[^3]

## 边界情况

**`n < 3`。** 外层循环 `range(n - 2)` 为空，函数返回 `[]`。约束 `3 <= nums.length` 把这种情况放在有效范围之外，但算法不写守卫也能干净处理。

**全零：`[0, 0, 0]`。** 锚点 `i = 0`（值 0）令 `target = 0`，立刻在 `(l = 1, r = 2)` 找到 `(0, 0)`，输出 `[0, 0, 0]`。两次内部跳跃都成了空操作，因为 `l = 2 > r = 1` 直接让内层循环退出。锚点 `i = 1` 触发规则（a），被跳过。算法恰好输出一个三元组，不是三个也不是五个。

**无解：`[1, 2, 3]`。** 提前退出的 `if nums[i] > 0: break` 在第一轮就触发。内层扫描从未运行。排序之后以 `O(1)` 返回 `[]`。

## 常见错误

**漏掉右指针去重。** 大多数提交写对了规则（a）和规则（b），忘了规则（c）。用 `[-2, -2, 0, 0, 2, 2]` 测一下：算法应当恰好输出一次 `[-2, 0, 2]`，而不是两次。

**用 `set(tuple(sorted([a, b, c])))` 去重。** 这个写法正确，而且在面试压力下觉得按位置跳跃太脆时，是个得体的兜底。代价是多 `O(n²)` 空间，外加 2-3 倍的常数因子变慢。资深面试官会把它评为「差一点」；三数之和这道题特意要考的，就是就地完成的那份去重。

**提前退出里写成 `>=` 而不是 `>`。** 写成 `if nums[i] >= 0: break` 会跳过全零用例。在 `[0, 0, 0]` 上第一个锚点的值为 0，规则触发，函数返回 `[]` 而不是 `[[0, 0, 0]]`。正确的规则是严格的。

**担心 `nums[l] + nums[r]` 整数溢出。** 这里不用担心。`|nums[i]| <= 10⁵` 时，两数之和放进 32 位有符号 `int` 还有余量。Java 和 C++ 实现都不需要转成 `long`。[LC 18 4Sum](https://leetcode.com/problems/4sum/) 就需要，因为它的约束范围更宽。

## 面试追问

**四数之和呢？** 那是 [LC 18](https://leetcode.com/problems/4sum/)。模板相同，多一层外层循环。固定两个锚点，在后缀上跑内层双指针扫描，目标是 `T - nums[i] - nums[j]`。外侧要做两层重复跳跃，再加上本篇原封不动的（b）（c）两条规则。总开销 `O(n³)`。推广到 k-Sum 就是 `O(n^(k-1))`。

**如果目标是最接近的和，而不是精确命中呢？** 那是 [LC 16 — 3Sum Closest](https://leetcode.com/problems/3sum-closest/)。算法相同，目标不同：找出和最接近给定 target 的三元组。三条重复跳跃规则全部删掉（你要的只是一个最接近的和，不是所有不同的三元组），把相等分支换成「计算 `|s - T|`，若优于当前最小值就更新」。开销同样是 `O(n²)`。

**那统计小于 target 的三元组个数呢？** 那是 [LC 259 — 3Sum Smaller](https://leetcode.com/problems/3sum-smaller/)。去重这套纪律消失了，因为数的是下标组合，不是值。当 `nums[i] + nums[l] + nums[r] < target` 时，由于数组有序，所有 `(l, l+1), (l, l+2), ..., (l, r)` 都满足条件，于是给计数加上 `r - l`，再把 `l` 前进一位。同样是 `O(n²)`。

**这道题的时间复杂度有下界吗？** 双指针的 `O(n²)` 与 Erickson 在 1999 年证明的线性决策树下界相符。[^2] 3Sum 在 word-RAM 模型下是否真的存在次平方算法，仍是开放问题；Pătrașcu 的 3SUM 硬度猜想，是几十个细粒度复杂度归约底下那个承重假设。[^4]

## 参考文献

[^1]: doocs/leetcode, "LC 15 3Sum." https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0015.3Sum/README_EN.md

[^2]: Wikipedia, "3SUM." https://en.wikipedia.org/wiki/3SUM

[^3]: Stack Overflow, "Leetcode #15: 3sum, avoiding duplicate," 2016. https://stackoverflow.com/questions/38909591/leetcode-15-3sum-avoiding-duplicate

[^4]: Pătrașcu, "Towards polynomial lower bounds for dynamic problems," STOC 2010. https://doi.org/10.1145/1806689.1806772
