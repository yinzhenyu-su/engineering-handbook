---
lc_id: 1
title: "两数之和"
slug: LC-001-two-sum
primary_chapter: part-1-linear-data-structures/03-hash-maps
star: true
difficulty: easy
date_updated: 2026-05-25
languages: [python, java, cpp, go]
editorial_widget: e-LC001-two-sum
---

# 两数之和

LC 1 是 LeetCode 上提交量最大的题目，也是技术面里最常见的开场题。算法本身只有四行。真正被考的是这四行之外的一切：先写出暴力解，说清它的复杂度，再提出优化，然后干净地写出来。就按这个顺序。

整节课的道理只有一句话。当内层循环在找一个特定值时，一张「值 → 下标」的哈希表能把这段循环换成一次 O(1) 的平均探测。剩下的内容全是操作顺序，以及那个重复值用例——它正好能暴露你把顺序写反了。

## 题目

给定整数数组 `nums` 和整数 `target`，返回两个数组位置，使这两个位置上的值之和等于 `target`。有且只有一对有效答案；同一位置不能用两次；返回顺序不限。约束规模落在简单题的区间：`2 <= nums.length <= 10⁴`，每个值与 target 都在 -10⁹ 到 10⁹ 之间。[^1]

LC 页面上的那句追问，才是这道题存在的全部理由：*你能做到比 O(n²) 更快吗？* 没有这一句，它就是一层嵌套循环，面试到这里就结束了。

| 输入 | 期望输出 | 考察点 |
|---|---|---|
| `[2, 7, 11, 15]`，target `9` | `[0, 1]` | 经典用例 |
| `[3, 2, 4]`，target `6` | `[1, 2]` | 跳过下标 0 |
| `[3, 3]`，target `6` | `[0, 1]` | 重复值要与**更早**出现的那一次匹配，而不是与自身匹配 |

最后一行会打断大多数人对优化版的第一次尝试。记住它。原文措辞见 [LC 1 Two Sum](https://leetcode.com/problems/two-sum/)。

## 解法 1：暴力

把题面直译成代码。取 `i < j`，检查 `nums[i] + nums[j] == target`，返回第一个命中的组合。

```python
# 暴力：O(n²) —— 这就是我们要替换掉的东西
def two_sum_brute(nums, target):
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []
```

内层边界是 `j > i`，不是 `j != i`。前一种写法把每个无序对恰好访问一次，并且从构造上保证两个位置不同；后一种需要事后再过滤一遍，传递出的信号是「我会给错算法打补丁」，而不是「我一开始就写对」。

时间 O(n²)，空间 O(1)。当 `nums.length = 10⁴` 时，最坏情况大约要跑 5 × 10⁷ 次比较，在 C++ 里属于临界，在 Python 里会在隐藏用例上超时。[^1] 把*「这是 O(n²)；追问想要更好的」*明确说出来，这句话是下一段对话需要的校准点。

## 解法 2：排序加双指针

这是第二直觉，也是值得点名的一次走错。数组有序的话，和就随指针位置单调变化：左指针右移让和变大，右指针左移让和变小。

```python
# 走错的一步：O(n log n)，但按 LC 1 的原题要求是错的
def two_sum_sort(nums, target):
    nums_sorted = sorted(nums)
    l, r = 0, len(nums) - 1
    while l < r:
        s = nums_sorted[l] + nums_sorted[r]
        if s == target:
            return [l, r]   # 错：这是排序后的下标，不是原始下标
        l, r = (l + 1, r) if s < target else (l, r - 1)
    return []
```

函数能跑，也确实返回一个长度为 2 的数组。但它没有解决 LC 1，因为 LC 1 要的是原始下标，而 `sorted` 把映射关系毁掉了。对 `[3, 2, 4]`、target 为 6，排序后的视图是 `[2, 3, 4]`，指针在排序后的位置 `(0, 2)` 上找到 `2 + 4 = 6`，函数返回 `[0, 2]`。正确答案是 `[1, 2]`。

修法是排序 `(value, original_index)` 元组。这样确实可行，而且这正是 [LC 167 Two Sum II](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) 的标准解法——那道题的输入本来就有序，而且「额外空间为常数」的规则不允许用哈希表。[^2] 放到 LC 1 上，它严格劣于哈希表：O(n log n) 对 O(n) 平均，还多出元组分配的开销。候选人提出排序、立刻发现下标被毁这个坑、随即转向哈希表，正好展示了他在权衡空间里的熟练度——这才是这个解法存在的意义：引出这场对话。

## 解法 3：一趟哈希表（最优）

暴力解的内层循环在找一个值：**补数** `target - nums[i]`。哈希表能用 O(1) 平均时间回答「我见过这个值吗，在哪个下标？」。把搜索换成一次探测，算法就从 O(n²) 塌缩到 O(n)。

这就是 Sedgewick & Wayne §3.4 里的教科书式归约：当双重嵌套循环的内层工作是在找一个键时，哈希表用一次探测替换掉整个内层循环。[^3] 随着 `i` 走过数组，只有两个问题要回答：

1. `target - nums[i]` 是否已经记录？是的话，配对下标就在表里；两个都返回。
2. 否的话，记录 `nums[i] -> i`，继续。

```python
def two_sum(nums, target):
    seen = {}                              # 值 -> 首次出现的下标
    for i, x in enumerate(nums):
        complement = target - x
        if complement in seen:
            return [seen[complement], i]   # 先查，后存
        seen[x] = i
    return []
```

**解法** — Python（[Java](./LC-001-two-sum/lc-1/sol.java) · [C++](./LC-001-two-sum/lc-1/sol.cpp) · [Go](./LC-001-two-sum/lc-1/sol.go)）

```python
# LC 1. Two Sum
from typing import Dict, List


def two_sum(nums: List[int], target: int) -> List[int]:
    """LC 1: return indices of two numbers in nums summing to target.

    One pass with a value -> first-seen-index map. For each x, look up the
    complement (target - x) BEFORE inserting x; the lookup-then-insert order
    is what prevents matching an element against itself on inputs like [3, 3].
    O(n) average time, O(n) space.
    """
    seen: Dict[int, int] = {}
    for i, x in enumerate(nums):
        complement = target - x
        if complement in seen:
            return [seen[complement], i]
        seen[x] = i
    return []
```

> **Interactive widget:** [两数之和（LC 1）—— 最优的一趟哈希表](../../../dsa/widgets/e-LC001-two-sum.yml) — 在网站上渲染；YAML 规范描述了各个关键帧。

*静态回退：`nums = [3, 3]`、target 为 6 时，第 0 次迭代拿补数 3 去空表里查（未命中），存入 `3 -> 0`。第 1 次迭代拿补数 3 去 `{3: 0}` 里查（命中），返回 `[0, 1]`。*

组件之所以演示 `[3, 3]`，是因为在 LC 给的三个例子里，只有它能验证「先查后存」这条不变式。另外两个例子在答案出现之前，表里已经装进三个不同的值，两种顺序都能跑通，把顺序颠倒过来的 bug 一直藏着。

### 为什么一趟胜过两趟

两趟的写法先建出完整的 `value -> index` 表，再扫一遍找补数。时间还是 O(n)，空间还是 O(n)。

```python
# 两趟写法 —— 等价，但要走两遍
def two_sum_two_pass(nums, target):
    seen = {nums[i]: i for i in range(len(nums))}
    for i, x in enumerate(nums):
        complement = target - x
        if complement in seen and seen[complement] != i:
            return [i, seen[complement]]
    return []
```

第 1 趟的字典推导会把前面的下标覆盖成后面的，所以在 `[3, 3]/6` 上表最后是 `{3: 1}`。`seen[complement] != i` 这个守卫，防止 `i = 0` 查到自己的补数时返回 `[0, 0]`。一趟版本是面试默认答案：一个循环而不是两个，内存一样，同样是 O(n) 上界，还少了一条要交代的不变式。

## 边界情况

**重复值，且两个都在答案里。** `[3, 3]`、target 6。约束禁止的是同一个**下标**用两次，不是同一个**值**用两次。在优化版里把查和存的顺序颠倒，就会产出 `[0, 0]`——同一个值跟自己配对，这是这个解法上最常见的 bug。

**负整数。** `[-1, -2, -3, -4]`、target -3。算法与符号无关，`target - x` 对任意有符号整数都成立。防御性的 `if x > target: continue` 过滤在负数输入上会出错，这种写法常常来自假设 target 非负的候选人。

**补数的整数溢出。** Python 的整数是任意精度，不是问题。Java、C++、Go 在多数平台上用 32 位 `int`。最坏情况下 `target - nums[i]` 是 `10⁹ - (-10⁹) = 2 × 10⁹`，仍在有符号 32 位 int 的上限（约 2.147 × 10⁹）以内，还有余量。[^1] 不需要转 `long`。

## 常见错误

**在 `[3, 3]/6` 上返回 `[0, 0]`。** 最经典的一个。候选人把 `seen[nums[i]] = i` 写在 `if complement in seen` *之前*，于是第 0 次迭代做补数检查时，表里已经有 `{3: 0}` 了。先查，后存。

**返回值而不是下标。** 在 `[2, 7, 11, 15]/9` 上候选人返回 `[2, 7]` 而不是 `[0, 1]`。修法是回去读题面——*「返回下标」*。

**用哈希*集合*而不是哈希*表*。** 集合的成员判断能回答「我见过这个值吗」，但把下标丢了。候选人最后给出的是对的值、错的下标。

**Java 在热路径上的装箱。** `HashMap<Integer, Integer>` 每次 `put` 和 `get` 都会给键和值自动装箱。在 LC 1 的最大输入下，最坏路径上就是 10⁴ 次分配：一次性提交无所谓，放到高 QPS 的生产环境里就很痛。生产上的修法是改用原始类型作键的 map，比如 Eclipse Collections 的 `IntIntHashMap`。面试官问「有没有更快的版本」，就是在提示这个升级。[^4]

## 面试追问

**如果输入有序呢？** 哈希表不再是合适的工具。两端相向的双指针能在 O(n) 时间、O(1) 空间内解决，空间上比哈希表更好。这就是 [LC 167 Two Sum II](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/)，而这个转向是整个指针—窗口家族里最有杠杆的模式信号：输入有序就能上双指针。[^2]

**如果有多对有效答案呢？** 只返回一对的版本可以提前短路，全对版本不行。要么命中后继续遍历、把每个匹配都追加进去；要么建一张 `value -> 下标列表` 的表，枚举 `seen[x]` 与 `seen[target - x]` 的笛卡尔积，并注意 `x == target - x` 时别重复计数。

**如果输入大到内存放不下呢？** 两种成本形态。数组在磁盘上：做外部排序，然后按 LC 167 的双指针扫描，两头流式读取。真正的流、不能回读：增量维护一个哈希集合，每来一个新元素就拿它的补数去查已有集合。流式版本本质上就是 [LC 170 Two Sum III](https://leetcode.com/problems/two-sum-iii-data-structure-design/)，用更快的 `add` 换更慢的 `find`，反过来也一样。

**那 Three Sum 呢？** 固定 `nums[i]`，在剩下的数组上以 `-nums[i]` 为 target 跑 Two Sum，得到 O(n²)。标准解法是排序后对每个锚点跑双指针：同样是 O(n²)，但靠跳过相同的锚点和相同的指针位置，去重很干净。从「无序、返回下标、哈希表」转向「有序、返回值、双指针、去重」，就是这个家族要教的东西。深入讲解见 [3Sum 题解](../../../dsa/editorials/LC-015-3sum.md)。

**最坏复杂度和期望复杂度差在哪？** 哈希表版 Two Sum 在对抗性冲突下最坏是 O(n²)，在简单均匀哈希下期望是 O(n)，也就是 CLRS 定理 11.1 给出的界。[^5] 生产环境里有缓解手段：Java 8 在链长到 8 时把链树化，把最坏情况压到 O(n log n)；Python 的 dict 用带扰动的开放寻址，能挡住最简单的冲突攻击。面试官在确认候选人是否分得清「被证明的界」和「实际交付的界」。

## 参考文献

[^1]: doocs/leetcode, "LC 1 Two Sum." https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0001.Two%20Sum/README_EN.md

[^2]: doocs/leetcode, "LC 167 Two Sum II — Input Array Is Sorted." https://github.com/doocs/leetcode/blob/main/solution/0100-0199/0167.Two%20Sum%20II%20-%20Input%20Array%20Is%20Sorted/README_EN.md

[^3]: Sedgewick and Wayne, *Algorithms*, 4th ed., Addison-Wesley, 2011, §3.4. https://algs4.cs.princeton.edu/34hash/

[^4]: walkccc, "LeetCode 0001." https://walkccc.me/LeetCode/problems/0001/

[^5]: Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press, 2022, Ch. 11.
