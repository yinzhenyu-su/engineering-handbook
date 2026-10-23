---
title: "复杂度的讨论"
description: "如何用一句话回答“时间复杂度是多少”：推导而不是背答案、时间与空间一起说、修饰词选对（最坏/平均/摊还），四个信号一次全给。"
slug: 03-complexity-discussions
part: 14
chapter: 3
difficulty: intermediate
prerequisites:
  - part-0-foundations/01-complexity-big-o
  - part-14-interview-framework/02-communicating
date_updated: 2026-05-25
languages: [python]
ladder:
  core: []
  stretch: []
---

# 复杂度的讨论

一场 45 分钟编程轮的第 34 分钟。你的代码能编译，测试用例全过，面试官往后一靠，问：“时间复杂度是多少？” 从现在开始，是三十秒的沉默。这三十秒就是这一页围绕的单位。

问题落地的那一刻，你其实已经知道那个界了。脑子里循环播放的那盘磁带是 “O(n)”、或者 “O(n log n)”、或者别的什么；问题在于，光说一句 “O(n)” 不算回答。面试官同时在听四件事，并据此给这个回答打分：你是推出来的还是背下来的，说得够不够紧，有没有把空间和时间一起报，修饰词给对了没有（最坏情况、期望、摊还，还是平均）。[^1] [计算复杂度与大 O](../part-0-foundations/01-complexity-big-o.md) 讲的是机械原理；下面这些是架在它上面的说法手册。

## 什么时候该说复杂度？

两次。一次是在你讲完思路、开始敲键盘之前（“我用一个哈希表，所以是 O(n) 期望时间、O(n) 空间”），一次是在代码能跑之后，用面试官会问的那种形式。写代码前先说这一句，能挡住最坏的结果：你写出的那个解法，如果一开始就把界说出口，面试官本来是会否掉的。代码跑完之后那第二遍，才是你复盘环节真正会被引用的那句。[^2]

主动报两次界还有个副作用。它说明你是从自己的代码里读出复杂度的，而不是从“做过类似题”的记忆里翻出来的。面试官一致地把“是 O(n²)，因为对同一个数组套了两层循环”排在那种听起来很自信、但毫无依据的“我觉得是 O(n log n)”前面，因为前者是面试官可以纠正的推理，后者只是猜。[^1]

## 回答听起来是什么样？

三句话。时间界、空间界、各自从哪来。往这个模板里填：

> [时间界] 时间，[空间界] 空间。时间来自 [主导的循环或递归]。空间来自 [辅助结构或递归深度]。

拿 Two Sum 填一遍，就是大多数读者都见过的那个“先暴力、再上哈希表”的重构：

```python
def two_sum_quadratic(nums, target):
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            if nums[i] + nums[j] == target:
                return [i, j]
    return []

def two_sum_linear(nums, target):
    seen = {}  # 值 -> 首次出现的下标
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
    return []
```

你要说出口的那段讨论，按顺序：

> “暴力解法是 O(n²) 时间、O(1) 额外空间。n² 来自那两层嵌套循环；下标是常数。我可以做得更好：带一张“值到下标”的哈希表，就是 O(n) 期望时间、O(n) 空间。我们是拿空间换掉了时间里的一个 n 因子，在任何现实的 n 上这都是对的交易。n ≤ 10⁵ 的题在 O(n²) 上会超时，而那个量级下，一张 n 条目的哈希表几乎不构成问题。”

这就是那个单位。说出来大约二十五秒，点了时间、点了空间、点了权衡，然后收住。面试官要么接受，要么追问下面两种可预测的追问之一，两种下文都覆盖了。

## 如果有两个参数呢？

那界就有两个字母。图上 V 个顶点、E 条边的 BFS 是 O(V + E)，不是 O(n)；合并 k 个有序链表、总长 n，是 O(n log k)，不是 O(n log n)。要避开的错误是那种偷懒的 O(n)，它把第二个参数藏进了“n 是输入规模”里。等面试官问“n 是什么？”，你就得花十秒钟，重新推一个你本来就已经有的界。

第一次就把哪个字母是什么说清楚。“V 是顶点数，E 是边数，BFS 每个顶点访问一次、每条边访问两次，所以是 O(V + E) 时间，O(V) 空间用来放 visited 集合和队列。” 如果题面给了你两个数组长度，就把两个名字都点出来：O(n + m)，其中 n 是 nums1 的长度，m 是 nums2 的长度。两个都用 n 表示，是个面试官分不清是笔误还是没理解的 bug。

## 期望、摊还、平均——该用哪个词？

三个词，三套论证，互不重叠。很多候选人把它们混着用，每混一次就丢一个校准分。[^1]

**期望（expected）** 是给哈希用的。在简单均匀哈希下，哈希表的插入和查找是 O(1) *期望*，当对手或者病态输入把所有键逼进同一个桶时，单次操作最坏 O(n)。[^3] Java 从第 8 版起把这个最坏情况降到每次查找 O(log n)：桶里的条目一到八个就转成红黑树，但前提是键实现了 Comparable；否则最坏情况还是 O(n)。[^4] 只说“O(1)”不说“期望”，等于主动招来那句“那最坏情况呢？”，而这句话你不如自己先答掉。

**摊还（amortized）** 是给容量翻倍的结构用的。Python 的 `list.append`、Java 的 `ArrayList.add`、C++ 的 `vector::push_back`、Go 的 `append`。绝大多数调用是 O(1)；碰上扩容那一次是 O(n)，因为每个元素都要被复制进一个翻倍大小的新数组。n 次调用累计下来，扩容的总工作量是 `1 + 2 + 4 + ... + n = 2n − 1`，摊到整段操作序列上，平均每次是 O(1)。[^5] 这是 *摊还*，不是平均；论证针对的是同一个结构上任意一个最坏序列，不是随机输入。[^6] 在延迟敏感的系统里，摊还界就不管用了，因为单次调用仍然可能飙到 O(n)，而用户感受到的正是那次飙升。[^5]

**平均（average）**（或者又写作 **期望**，看你读的是谁的教材）是给随机化算法用的。快排在有随机枢轴或三数取中时，期望运行时间是 O(n log n)；如果取末元素作枢轴、输入又有序，最坏情况是 O(n²)。这里的期望是对算法抛硬币取的，不是对随机输入取的。随机化让这个界在对抗性数据面前依然成立。[^3] 哈希表的 *期望* O(1) 和快排的 *期望* O(n log n) 是同一种期望，都是在算法的随机性上取的。

口头规则：哈希用 *期望*，容量翻倍用 *摊还*，随机化算法用 *平均* 或 *期望*。三个词，三种配对，映射建好之后就自动选出来了。

## 如何回应“能不能做得更好？”

这个问题很少是在要一个更紧的界；它是在检查你知不知道存在这么一个界。三种诚实的回答能覆盖大多数情况。

如果比较排序的下界正摆在台面上，而你的界正好等于它，就直说：“一般比较模型下不行；n log n 就是比较排序的下界。如果键是范围有界的整数，可以用计数排序或基数排序绕过去，但常数决定了这个交换只在 n 很大时才划算。”[^3] 点出下界，把对话从“找一个更快的算法”挪到“放松输入模型”，后者才是有产出的下一步。

如果你是用空间换的时间，而面试官正在找对称的那笔交易，就把代价报出来：“我可以用 [原地修改 / 双指针 / 旋转的三次反转技巧] 把空间降到 O(1)，但那会改动输入——这可以接受吗？” 动手改之前先问，能省掉一个很难收回的错误假设。

如果你确实不知道还能不能更好，就坦白说：“我看不出更紧的界；你心里有一个吗？” 一句诚实的“我不知道，我漏了什么？” 比五秒钟的卡壳好听得多。

## 常见陷阱

> [!WARNING]
> **只说时间不说空间。** 复杂度讨论里丢分最多的第一名，就是候选人说了“O(n)”，然后等着面试官追问“那空间呢？”才报辅助内存。[^1] 把“时间 AND 空间”当成一个词组背下来。辅助空间包括哈希表、结果数组、递归栈，以及算法申请的任何临时结构；不包括输入本身。

> [!WARNING]
> **混淆最好、平均、最坏情况。** 如果算法在不同情况下的界不同（快排、哈希表、BST、布隆过滤器），就主动把两个都报出来：“快排是 O(n log n) 期望，O(n²) 最坏。” 让面试官来问“那最坏情况呢？”，等于白白丢掉一个信号——你本来知道这个不对称是存在的。

> [!WARNING]
> **循环里藏着库函数调用。** 一个带 `if substring in s:` 的紧凑循环不是 O(n)；子串扫描本身就是 O(n)，所以总共是 O(n·m)。`list.remove(x)`（O(n)）、`x in list`（O(n)）、切片（要复制，O(k)）是同一个陷阱。对每个方法调用，先问清楚库在干什么，再给你包着它的那个循环贴 Big-O 标签。

> [!WARNING]
> **把递归当免费。** 递归版的原位快排平均是 O(log n) 辅助空间，不是 O(1)。递归栈是真实内存：每一帧都要放参数和返回地址。改写成迭代，渐近空间是一样的（显式栈替代了调用栈）；改成迭代真正的好处是躲开 Python 默认 1000 帧、Java 大约 10⁴ 帧的上限，而不是渐近空间。

## 后续衔接

整套脚本就这些。把回答的骨架练成反射，是[模拟面试流程](./04-mock-interview-process.md)讲的事。相邻那一章[面试中的沟通](./02-communicating.md)把复杂度讨论看成更大的“该说出口什么”问题里的一片，而[常见面试陷阱](./05-common-pitfalls.md)汇总了四类面试场景里的高频错误。

## 参考文献

[^1]: codeintuition.io, "How to Analyze Your Code's Complexity in an Interview" (retrieved 2026-05-19), <https://www.codeintuition.io/blogs/time-and-space-complexity-interview>.

[^2]: Lerner, A. and Holtz, D., "We analyzed thousands of technical interviews on everything from language to code style. Here's what we found," interviewing.io blog (post date circa 2016-2019, retrieved 2026-05-20), <https://interviewing.io/blog/we-analyzed-thousands-of-technical-interviews-on-everything-from-language-to-code-style-here-s-what-we-found>.

[^3]: Cormen, T.H., Leiserson, C.E., Rivest, R.L., Stein, C., *Introduction to Algorithms*, 4th ed., MIT Press, 2022, ISBN 978-0-262-04630-5.

[^4]: Stack Overflow, "The worst case time complexity for HashMap after Java 8 is still O(n) not O(log n)?" (April 2024), <https://stackoverflow.com/questions/78295005/>.

[^5]: Tarjan, R.E., "Amortized Computational Complexity," *SIAM Journal on Algebraic and Discrete Methods*, Vol. 6, No. 2, April 1985, pp.

[^6]: *Amortized analysis*, Wikipedia (retrieved 2026-05-19), <https://en.wikipedia.org/wiki/Amortized_analysis>.
