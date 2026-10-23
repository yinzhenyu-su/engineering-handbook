---
title: "比较排序 II：快速排序、分区与生产级混合排序"
description: "归并排序与快速排序：分治、分区、稳定性，以及为什么生产库里的排序都是混合算法。"
slug: 04-comparison-sorts-2
part: 2
chapter: 4
difficulty: intermediate
prerequisites:
  - part-2-search-sort/03-comparison-sorts-1
  - part-1-linear-data-structures/00-arrays
  - part-0-foundations/01-complexity-big-o
date_updated: 2026-05-24
languages: [python, java, cpp, go]
canonical_test: LC-912
widgets: [w-05-sorting-visualizer, w-06-quicksort-partition]
ladder:
  core: [LC-905, LC-2161, LC-324]
  stretch: [LC-462, LC-922]
  star: LC-2161
---

# 比较排序 II：快速排序、分区与生产级混合排序

Hoare 于 1961 年在 *Communications of the ACM* 上发表了 Algorithm 64，给世界留下六行伪代码，外加一句大白话：选一个枢轴（pivot），重排数组让比它小的元素落在左边、比它大的落在右边，然后对两半递归。六行。六十五年过去，C 标准库里的那个函数还叫 `qsort`，C++ 的 `std::sort` 依然建在 Hoare 的这个原语上，Java 的基本类型数组排序是它的双轴变体。这套骨架之所以没被淘汰，是因为分区这一步真的很便宜：每个递归层一趟 O(n) 扫描，事后不用合并，不用辅助缓冲区。在随机输入上，快速排序的常数因子胜过其他任何比较排序。

问题在于，同样这六行有一个 Theta(n^2) 的最坏情况。[^1] 拿教科书版快排去跑一个已经排好序的数组、枢轴取最后一个元素，它会走成一条线性递归链，每一层都分出一个「一个元素 + 其余全部」的切分。这个病在生产里腐蚀性太强，以至于 Tim Peters 在 2002 年写了 Timsort 来替换 CPython 的排序，[^2] David Musser 在 1997 年提出内省排序（introsort）来修 C++ 的排序，[^3] Java 团队则在 JDK 7 里采纳了 Vladimir Yaroslavskiy 的双轴方案。本章两半都讲：算法本身，以及裹在它外面的生产级脚手架。

这一对章节最终要收敛到一句你该能背下来的话。*归并排序稳定、最坏情况保证 O(n log n)，但要分配 O(n) 辅助空间；快速排序不稳定、期望 O(n log n)，但原地排序。* 这句话就是「这里为什么用归并不用快排」的答案，也是面试官质疑你的选择时想听到的东西。

## 算法 A：三数取中的 Lomuto 分区快速排序

教快排最清晰的路子是先讲分区，因为剩下的全是你已经会的递归。从一个六元素数组、目标枢轴值 6 开始：

```text
arr = [5, 2, 4, 1, 3, 6]
pivot = 6 (the rightmost element)
```

分区之后，所有小于等于 6 的元素都在前缀里，所有大于 6 的元素都在后缀里。枢轴落在它最终排好序的位置上。一趟扫描，两个下标。

这个版本叫 Lomuto 分区（CLRS 里干脆就叫 PARTITION），七行就能读完。[^4] 两个下标：`i` 是「小于等于枢轴」前缀的边界；`j` 从 `lo` 扫到 `hi - 1`。当 `arr[j] <= pivot` 时，算法把 `i` 往前推一格，交换 `arr[i]` 和 `arr[j]`，前缀扩一个格子。当 `arr[j] > pivot` 时什么都不做，这个格子留在「大于枢轴」的中间带里，随着 `j` 一起变长。扫描结束后再做一次交换，把枢轴放到下标 `i + 1`，正好夹在两个带之间。

让算法正确的正是循环不变式：每轮迭代时，`arr[lo..i]` 里是 `<= pivot` 的元素，`arr[i+1..j-1]` 里是 `> pivot` 的元素，`arr[j..hi-1]` 尚未检查。初始化 `i = lo - 1` 让前缀为空；`j` 在区间上迭代；不变式在每一步都成立。

> **交互组件：** [快速排序分区（Lomuto，逐步演示）](../../../dsa/widgets/w-06-quicksort-partition.yml) —— 在网站上渲染；YAML 规格描述了关键帧。

*静态版：`[5, 2, 4, 1, 3, 6]` 上 Lomuto 扫描的 16 步动画，展示 `i` 和 `j` 如何前进、前缀如何增长、枢轴最终落在下标 5。*

### 为什么最直白的写法在有序输入上会爆炸

把教科书的 PARTITION 过程塞进递归外壳里，你会得到这个：

```python
# 暴力解法：能跑，但在有序输入上会炸。
def quicksort_naive(arr, lo, hi):
    if lo >= hi:
        return
    pivot = arr[hi]                          # 取最后一个元素，完全没过脑子
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    p = i + 1
    quicksort_naive(arr, lo, p - 1)
    quicksort_naive(arr, p + 1, hi)
```

把它提交到 LC 912，那个隐藏的 5 万元素有序输入用例会超时。原因很机械。在有序数组上取 `pivot = arr[hi]`，每次比较 `arr[j] <= pivot` 都成立（枢轴就是数组最大值），于是 `i` 每轮都前进，最后停在 `hi - 1`。分区返回 `hi`。下一层递归处理 `arr[lo..hi-1]`，一个规模 `n - 1` 的子区间，然后这个模式重复。递归深度变成 `n`。总工作量变成递推 `T(n) = T(n - 1) + Theta(n)`，解得 `Theta(n^2)`。CLRS 习题 7.2-3 明确点了这个触发条件：「数组 A 含互不相同的元素且按降序排好」；在末元素取枢轴的 Lomuto 下，升序有序输入也会同样爆炸。[^4] 逆序输入更糟：分区在每一层除了线性递归还要做 `n - 1` 次交换。

修法是别再让输入决定枢轴。有两个办法都管用，而且都是正确性修复，不是优化。CLRS 的 RANDOMIZED-PARTITION 在分区前把 `arr[hi]` 与 `arr[random(lo, hi)]` 交换；不管输入顺序如何，期望运行时间都变成 `Theta(n log n)`，因为对手看不到随机硬币就选不出最坏输入。[^4] Sedgewick 的三数取中是确定性的替代方案：取 `arr[lo]`、`arr[mid]`、`arr[hi]` 三个样本，把它们的中位数放到 `arr[hi]`，再分区。[^5] 两种选择都把有序输入这个病态变成现实中任何输入都触发不了的事件；三数取中的常数因子略好（期望比较次数从大约 `1.39 n log_2 n` 降到 `1.19 n log_2 n`，代价是每次分区多三次比较），本章把它当作标准参考实现来教。

### 教学参考实现

```python
def _quicksort(arr, lo, hi):
    while lo < hi:
        # 三数取中：把 median(arr[lo], arr[mid], arr[hi]) 放到 arr[hi]。
        mid = lo + (hi - lo) // 2          # 相比 (lo + hi) // 2 不会溢出
        if arr[mid] < arr[lo]:
            arr[lo], arr[mid] = arr[mid], arr[lo]
        if arr[hi] < arr[lo]:
            arr[lo], arr[hi] = arr[hi], arr[lo]
        if arr[mid] < arr[hi]:
            arr[mid], arr[hi] = arr[hi], arr[mid]
        # arr[hi] 现在持有中位数；拿它当 Lomuto 枢轴。
        p = _lomuto_partition(arr, lo, hi)
        # 对较小的一侧递归，对较大的一侧循环（Sedgewick）。
        if p - lo < hi - p:
            _quicksort(arr, lo, p - 1)
            lo = p + 1
        else:
            _quicksort(arr, p + 1, hi)
            hi = p - 1


def _lomuto_partition(arr, lo, hi):
    pivot = arr[hi]
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    return i + 1
```

**解法** —— Python（[Java](./04-comparison-sorts-2/lc-912/sol.java) · [C++](./04-comparison-sorts-2/lc-912/sol.cpp) · [Go](./04-comparison-sorts-2/lc-912/sol.go)）

```python
# LC 912. Sort an Array
# 三数取中 + Lomuto 分区的快速排序。
from typing import List


def sort_array(nums: List[int]) -> List[int]:
    """LC 912: sort and return a new list. Input is not mutated."""
    arr = list(nums)
    if len(arr) > 1:
        _quicksort(arr, 0, len(arr) - 1)
    return arr


def _quicksort(arr: List[int], lo: int, hi: int) -> None:
    """In-place quicksort on arr[lo..hi] inclusive."""
    while lo < hi:
        # 三数取中：把 median(arr[lo], arr[mid], arr[hi]) 放到 arr[hi]。
        mid = lo + (hi - lo) // 2
        if arr[mid] < arr[lo]:
            arr[lo], arr[mid] = arr[mid], arr[lo]
        if arr[hi] < arr[lo]:
            arr[lo], arr[hi] = arr[hi], arr[lo]
        if arr[mid] < arr[hi]:
            arr[mid], arr[hi] = arr[hi], arr[mid]
        # arr[hi] 现在持有三者中位数；拿它当 Lomuto 枢轴。
        p = _lomuto_partition(arr, lo, hi)
        # 对较小的一侧递归，对较大的一侧循环（Sedgewick 的尾调用修复）。
        if p - lo < hi - p:
            _quicksort(arr, lo, p - 1)
            lo = p + 1
        else:
            _quicksort(arr, p + 1, hi)
            hi = p - 1


def _lomuto_partition(arr: List[int], lo: int, hi: int) -> int:
    """CLRS 4th ed. PARTITION: pivot is arr[hi]. Returns final pivot index."""
    pivot = arr[hi]
    i = lo - 1
    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    return i + 1
```

有三个细节配得上它们占的行数。一个都不是可选项，漏掉任何一个都会带着真 bug 上线。

**`mid = lo + (hi - lo) // 2`** 躲开了 Joshua Bloch 2006 年在 `java.util.Arrays.binarySearch` 里记录的整数溢出，那个 bug 躺了九年没被发现。[^6] 在有符号 32 位算术里，数组超过 2^30 个元素后 `lo + hi` 就溢出，除法结果保持负数，递归跑出边界。在无界整数上这两种写法等价；在有界整数类型上不等价。把安全写法变成肌肉记忆。

**Sedgewick 的尾调用消除** —— 对较小的一侧递归、对较大的一侧循环 —— 即使在最坏情况下也能把递归栈限制在 O(log n)。朴素的递归快排在对抗性输入上要用 O(n) 个栈帧；Python 在深度 1000 时抛 `RecursionError`，Java 根据每帧大小不同会在 2,000 到 10,000 之间的某处抛 `StackOverflowError`。`while lo < hi` 配合「先处理较小一侧」的写法，处理较大子区间时不再消耗栈帧，即使分区行为很糟，栈深度也只和输入的 log 成正比。省掉这个写法，5 万元素的最坏输入不是超时而是直接崩，调起来更难。

**三数取中把枢轴放到 `arr[hi]`**，不是 `arr[mid]`。这是 Lomuto 特有的约定：`_lomuto_partition` 按其定义就是从 `arr[hi]` 读枢轴。那三行比较把 `(arr[lo], arr[mid], arr[hi])` 排好序，让 `arr[hi]` 最终持有中位数。换个放法，分区的不变式从第一趟扫描起就是错的。

四份实现共用同一套「三数取中 + Lomuto」骨架；唯一因语言而异的让步是开头的防御性拷贝，让函数不改调用方的缓冲区。四份全过八例标准测试，包括有序输入 `[1, 2, 3, 4, 5]` 和逆序 `[5, 4, 3, 2, 1]` —— 这两个用例在朴素的末元素取枢轴 Lomuto 下会触发 Theta(n^2)。

### 实例推演：在 `[5, 2, 4, 7, 1, 3, 2, 6]` 上跑三数取中 Lomuto

这个标准输入就是组件里 panel-a 在归并排序下动画演示的那八个元素，所以并排拖动两个面板就能看出结构上的对比：归并排序自顶向下分解再合并回来；快速排序原地分区、递归，不合并。

第一次分区取 `(arr[0]=5, arr[3]=7, arr[7]=6)` 的三数取中得 `6`，把中位数换到 `arr[7]`，然后跑 Lomuto。枢轴 6 落在下标 6；左带 `[5, 2, 4, 1, 3, 2]` 是所有 `<= 6` 的元素；右带只有 `[7]`。递归往下走三层，接下来三个子问题分别选枢轴 4、2、1。两个重复的 `2` 在第 2 层一起走；`(2, 2, 3)` 的三数取中选 `2` 当枢轴，分区把第二个 `2` 落在第一个旁边。快速排序不稳定：它们在输出里的相对顺序可能和输入不同。

> **交互组件：** [排序可视化（归并排序 + 快速排序）](../../../dsa/widgets/w-05-sorting-visualizer.yml)（面板 `panel-b`）—— 在网站上渲染；YAML 规格描述了关键帧。

*静态版：在 `[5, 2, 4, 7, 1, 3, 2, 6]` 上跑三数取中 Lomuto 算法的 22 步轨迹，最终得到 `[1, 2, 2, 3, 4, 5, 6, 7]`，每次分区时递归树的深度都可见。*

组件里有两个预设是本章特意要你对比的。「有序输入 + 朴素末元素取枢轴 Lomuto」预设对 `[1, 2, 3, 4, 5, 6, 7, 8]` 跑教科书版 PARTITION，走出 36 步轨迹，算法一路走成退化的七层线性递归。「有序输入 + 三数取中」预设在同一个数组上 22 步就跑完。切换预设，二次爆炸就体现在步数上：同样的数据、同样的算法外壳、同样的分区方案。只有枢轴规则变了，Theta(n^2) 就成了 Theta(n log n)。

## 算法 B：生产环境的混合排序

教科书算法不是你标准库里装的那个。Hoare 那篇论文之后五年，Bentley 和 McIlroy 在 1993 年的 *Engineering a Sort Function* 里记录了 BSD `qsort` 一脉要活过生产所必需的实用补丁：中等子数组用三数取中，大子数组用 ninther（三个中位数中的中位数），键值大量重复的数组用三路「胖分区」，小子数组用插入排序截断。[^7] 这四条加上一个兜底，就是现代每一个标准库的样子。

### 内省排序：带堆排序安全网的快速排序

David Musser 1997 年的内省排序就是 C++ STL 在 libstdc++ 和 libc++ 里以 `std::sort` 交付的那个算法。[^3] 它跑三数取中快排，但在下沉过程中记录递归深度；深度超过 `2 log_2 n` 就中止这次递归调用，对剩余子区间跑堆排序。这个混合体把快排的平均情况常数因子和堆排序的最坏情况保证拼在一起，任何输入都是 `Theta(n log n)`，常数还是快排那种典型常数。内省排序就是「C++ 的 `std::sort` 凭什么保证 O(n log n) 而快排不能？」的生产答案。答案是：它不是快排。

堆排序之所以是合适的兜底，是因为它最坏 `Theta(n log n)`、原地（`O(1)` 辅助空间）、对输入顺序不敏感。它的缓存行为比快排差，这恰恰是内省排序把它当后手而不是默认的原因。绝大多数调用根本碰不到深度阈值；少数本来会退化成二次的，被堆排序救了回来。下一章 [堆排序与 n log n 下界](../../../dsa/part-2-search-sort/05-heap-sort.md) 会证明这个界，并展示堆排序的机制。

### Timsort：稳定、自适应、世界上用得最多的比较排序

2002 年 Tim Peters 为 CPython 写了一个排序例程，此后它成了世界上部署量最大的比较排序。Timsort 出现在 Python 的 `list.sort()` 和 `sorted()` 里，从 Java 7（2011 年）起出现在 Java 的 `Arrays.sort(Object[])` 里，2009 年起出现在 Android Java 里。[^2] 这个算法先找输入中的自然游程（runs，即最长的已排序子序列），用插入排序把短游程延长到最小长度，再按游程长度比例上的不变式从一个栈里合并游程，那些不变式约束了最坏合并代价。最坏 Theta(n log n)，在已排序输入上最好 Theta(n)，并且稳定。

Timsort 的 Tim Peters 设计笔记至今还在 CPython 源码的 `Objects/listsort.txt` 里，如果你想看生产级排序工程师到底在争什么，值得读一遍。[^2] 相关的那段：「它在很多种部分有序的数组上有超自然的性能（需要的比较次数少于 lg(N!)，最少只要 N-1 次），而在随机数组上和 Python 之前高度调优的 samplesort 混合体一样快。」赢的地方在于真实世界的数据几乎总是部分有序的。日志文件带着成批的乱序到达；提交列表从多个 feed 汇总而来；新闻文章的发布时间戳；表格导入。最坏情况下的均匀输入是学术虚构；部分有序的输入才是生产代码真正见到的。

2015 年，Stijn de Gouw 和同事用 KeY 定理证明器验证 Timsort 的游程合并逻辑，发现 Java 的移植版在 `merge_collapse` 里带了一个微妙的 bug，在一个刻意构造的输入上会撑爆待合并游程的栈。[^8] Java 团队在 OpenJDK 8u80 里修了这个实现；CPython 保留了原版，因为没有哪台现实中的机器能装下大到触发这个 bug 的数组。这个教训就是本章贯穿的主题。生产级排序例程不是无视复杂度的借口；它们是一代工程师拒绝无视复杂度的结果。

### 双轴快排：Java 的基本类型数组排序

Vladimir Yaroslavskiy 2009 年的双轴快排就是 Java 7+ 里 `Arrays.sort(int[])`（以及 `long[]`、`double[]` 等基本类型重载）装的东西。[^9] 这个算法选两个枢轴 `p1 <= p2`，把数组分成三个带 `[< p1, p1..p2, > p2]`，然后分别递归。性能收益由缓存驱动：三个大致等长的子区间比单枢轴的两个子区间更贴合现代 CPU 的缓存层次，在随机整数数组上常数因子比单轴快排低约 10%。Go 的标准库走了同一个方向，`slices.Sort` 用的是 pdqsort（pattern-defeating quicksort，模式击败快排）。

### 归并排序与快速排序之间如何取舍

从这个权衡推出的库设计经验法则是**对象用归并，基本类型用快排**。对象带有可被观察的次要键，所以稳定性重要；基本类型是没有可区分副本的值，稳定性无关。Java 7+ 在标准库里把这个分界写得很明确：

| Java API | 算法 | 稳定？ | 原因 |
|---|---|---|---|
| `Arrays.sort(int[])` 及其他基本类型重载 | 双轴快排（Yaroslavskiy 2009） | 否 | 基本类型没有可观察的次要键；缓存局部性更重要 |
| `Arrays.sort(Object[])` | Timsort | 是 | 对象可能带次要键；多键排序需要稳定性 |

C++ 的选择不同：`std::sort` 是内省排序（不稳定，为缓存局部性优化），稳定性要通过 `std::stable_sort` 显式选择（通常是归并排序或块排序）。Go 的 `slices.Sort` 是 pdqsort（不稳定）；稳定性通过 `slices.SortStableFunc` 显式选择。Python 的 `list.sort` 任何时候都是 Timsort（稳定）。

知道自己语言的默认行为，是在多键排序上信任标准库的前提。从一个 Python 代码库换到 Go 而不标注稳定性的变化，就是多键 bug 静默上线的方式。

## 开销

| 变体 | 最好时间 | 平均时间 | 最坏时间 | 空间（辅助） | 稳定？ |
|---|---|---|---|---|---|
| Lomuto，末元素取枢轴 | Theta(n log n) | Theta(n log n) | Theta(n^2) | O(n) | 否 |
| Lomuto，三数取中 | Theta(n log n) | Theta(n log n) | Theta(n^2)（罕见） | O(log n) | 否 |
| RANDOMIZED-QUICKSORT（CLRS） | Theta(n log n) | Theta(n log n) | 期望 Theta(n log n) | O(log n) | 否 |
| Hoare，双指针 | Theta(n log n) | Theta(n log n) | Theta(n^2) | O(log n) | 否 |
| 三路（荷兰国旗） | k=1 时 Theta(n) | Theta(n H) | Theta(n log n) | O(log n) | 否 |
| 内省排序 | Theta(n log n) | Theta(n log n) | Theta(n log n) | O(log n) | 否 |
| Timsort | Theta(n) | Theta(n log n) | Theta(n log n) | Theta(n) | 是 |
| 双轴快排 | Theta(n log n) | Theta(n log n) | Theta(n^2) | O(log n) | 否 |

朴素递归 Lomuto 上那个 Theta(n) 辅助空间，就是对抗性输入下的调用栈；Sedgewick 的尾调用修复把它限制在 O(log n)。内省排序的堆排序兜底把*时间*限制在 Theta(n log n)，在不牺牲平均性能的前提下保证了最坏界。RANDOMIZED-QUICKSORT 的期望时间界来自指示变量分析：期望比较次数是 `2(n+1) H_n - 4n ≈ 2n ln n ≈ 1.39 n log_2 n`，在 CLRS §7.4.2 里推导。[^4] 三路（Bentley-McIlroy 胖分区）降到 Theta(n H)，其中 H 是键分布的香农熵，对于 k 个不同键上界为 `log_2 k`；这在比较排序里可证明是熵最优的。[^5]

## 常见陷阱

> [!CAUTION]
> **朴素的末元素取枢轴 Lomuto 在有序输入上是 Theta(n^2)。** 这是本章的头号失效模式。读者照抄教科书版 PARTITION 过程，跳过随机化或三数取中那一步，提交到 LC 912。提交在隐藏的 5 万元素有序输入用例上超时。修法是随机化**或**三数取中；两者都是正确性修复，不是优化。

> [!WARNING]
> **Lomuto 分区在全等输入上，即使做了三数取中也会塌成 Theta(n^2)。** 每个元素都满足 `arr[j] <= pivot`，于是 `i` 每轮都前进，分区函数返回 `hi`，产生一个 (n-1) 元素的左侧和一个空的右侧。同样的输入换成 Hoare 分区（它会在两端对等于枢轴的元素停下）就会产生均衡切分。重复元素多的数组要用**三路分区**（Dijkstra 的荷兰国旗）；只有偶尔重复时，Hoare 分区就够了。[^5]

> [!WARNING]
> **Hoare 分区的约定很刁钻。** 返回 `i` 而不是 `j`，枢轴就落在错误的递归区间里；先自增再判断时用 `<=` 而不是 `<`，两个指针会停在错误的位置；从上半区取枢轴会破坏递推的不变式。Lomuto 是合适的教学参考，正因为它绕开了这片雷区。生产里要用 Hoare，就从经过验证的参考实现里抄（Sedgewick 的 `Quick.java` 或 Bentley-McIlroy 的 `qsort`），别凭记忆复现。

> [!WARNING]
> **快速排序不稳定。** 「先按主键、再按次键」的多键排序，期望主键相等的记录在次键排序后保持输入顺序；在快排下，第二次排序会打乱同一个键内部的顺序。C++ 的 `std::sort` 是内省排序，也不稳定；要稳定必须显式用 `std::stable_sort`。Python 的 `list.sort` 和 Java 的 `Arrays.sort(Object[])` 是 Timsort，默认稳定；换成基于快排的排序，会静默破坏依赖稳定性的代码。

> [!WARNING]
> **没有 Sedgewick 尾调用修复的递归，在对抗性输入上会栈溢出。** 三数取中快排碰上 Bentley-McIlroy 的「antiquicksort」序列（专门为击败三数取中而设计）可以递归到深度 n。Python 抛 `RecursionError`；Java 抛 `StackOverflowError`；原生代码段错误。修法是「对较小一侧递归、对较大一侧循环」这个写法；配合三数取中，期望和最坏情况的栈深度都是 O(log n)。

## 题目梯度

CORE 跨了简单、中等、中等三档，因为 LeetCode 上经典的困难级「分区 + 快排」题归相邻章节所有：LC 215 归 [快速选择](../../../dsa/part-2-search-sort/07-quickselect.md)，LC 75 归 [双指针，相向而行](../../../dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md)。本章交付的是最干净的分区练习，能演示这个模式又不从邻章抢题。

### 核心（全做才算掌握本章）

- [LC 905 — Sort Array By Parity](https://leetcode.com/problems/sort-array-by-parity/) [简单] • 分区
- [LC 2161 — Partition Array According to Given Pivot](https://leetcode.com/problems/partition-array-according-to-given-pivot/) [中等] • 分区 ★
- [LC 324 — Wiggle Sort II](https://leetcode.com/problems/wiggle-sort-ii/) [中等] • 快速选择加分区

### 进阶（可选）

- [LC 462 — Minimum Moves to Equal Array Elements II](https://leetcode.com/problems/minimum-moves-to-equal-array-elements-ii/) [中等] • 通过分区做快速选择
- [LC 922 — Sort Array By Parity II](https://leetcode.com/problems/sort-array-by-parity-ii/) [简单] • 分区

## 后续衔接

下一章 [堆排序与 n log n 下界](../../../dsa/part-2-search-sort/05-heap-sort.md) 会证明为什么没有任何基于比较的算法能在最坏情况下优于 Theta(n log n)，并展示第三个达到这一界、只用 O(1) 辅助空间的 O(n log n) 排序；也就是内省排序拿来兜底的那个堆排序。之后 [线性时间排序：计数、基数、桶排序](../../../dsa/part-2-search-sort/06-linear-sorts.md) 讲那些彻底离开比较模型、绕过下界的算法。这里确立的分区原语会在 [快速选择](../../../dsa/part-2-search-sort/07-quickselect.md) 里再次出现，那里一次分区加单侧递归给出期望 O(n) 的选择。三路分区的变体则成为 [双指针，相向而行](../../../dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md) 的基底，那里 Dijkstra 的荷兰国旗把它特化到 K=3 的情形，对应 LC 75 的层次。

## 参考文献

[^1]: C. A. R. Hoare, "Algorithm 64: Quicksort," *Communications of the ACM*, 4(7):321, July 1961.

[^2]: Tim Peters, design notes for Timsort, in CPython source `Objects/listsort.txt`, ratified for CPython 2.3 in 2002. <https://github.com/python/cpython/blob/main/Objects/listsort.txt>.

[^3]: David R. Musser, "Introspective Sorting and Selection Algorithms," *Software: Practice and Experience*, 27(8):983-993, August 1997.

[^4]: Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press, 2022, Chapter 7 "Quicksort".

[^5]: Robert Sedgewick and Kevin Wayne, *Algorithms*, 4th ed., Addison-Wesley, 2011, §2.3 "Quicksort". Online at <https://algs4.cs.princeton.edu/23quicksort/>.

[^6]: Joshua Bloch, "Extra, Extra — Read All About It: Nearly All Binary Searches and Mergesorts are Broken," *Google Research blog*, June 2, 2006. <https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/>.

[^7]: Jon L. Bentley and M. Douglas McIlroy, "Engineering a Sort Function," *Software: Practice and Experience*, 23(11):1249-1265, 1993.

[^8]: Stijn de Gouw, Jurriaan Rot, Frank S. de Boer, Richard Bubel, Reiner Hähnle, "OpenJDK's `java.utils.Collection.sort()` is broken: The good, the bad and the worst case," CAV 2015. Companion bug tracker entry: <https://bugs.python.org/issue23515>.

[^9]: Vladimir Yaroslavskiy, "Dual-Pivot Quicksort," 2009. Detailed analysis: Sebastian Wild and Markus Nebel, "Average case analysis of Java 7's dual pivot quicksort," European Symposium on Algorithms, 2012, <https://arxiv.org/abs/1310.7409>.
