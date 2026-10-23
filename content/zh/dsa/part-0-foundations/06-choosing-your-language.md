---
title: "选择你的面试语言"
description: "如何为技术面试挑一门语言：权衡、每门语言要你在解题时间上付出什么代价，以及一棵决策树。"
slug: 06-choosing-your-language
part: 0
chapter: 6
difficulty: beginner
prerequisites: [00-how-to-use, 05-language-idioms]
date_updated: 2026-05-24
languages: [python, java, cpp, go]
ladder:
  core: []
  stretch: []
---

# 选择你的面试语言

周五上午 10 点，你有一场 Meta 电话面试。工作时你写 Java，但为了让语法短一点，你已经用 Python 刷了一个月 LeetCode。现在是周三晚上。你有九十秒决定周五打开编辑器时用哪门语言，而一百小时的练习已经投在了错误的方向上。选吧。

这才是真问题。网上把这件事当成一场淘汰赛，总想选出唯一赢家；数据说的不是这样。interviewing.io 对约 3,000 场有记录的技术面试做了定量分析，发现通过率与语言选择之间没有统计显著差异，只有一个例外，p = 0.037：用 Java 面试的候选人，在公司技术栈本身就是 Java 时通过率更高。[^1] 他们的直接结论是："顺手用好你惯用的语言，永远胜过拿一门听起来很高级、你却不熟的语言炫技。"[^1]

所以本章不颁奖。它只讲：什么时候该换、换到多晚算太晚，以及坑表里哪两行解释了一半的"我本地跑通了"故事。

## 该选哪门语言

如果你手里没有已经用得很顺的语言，默认选 **Python**。前 Meta 资深工程师、Blind 75 作者 Yangshun Tay 推荐 Python、C++、Java 或 JavaScript，他给出高层语言更优的经典理由："像 Python 或 Java 这样的高层语言自带标准库函数与数据结构，能让你更轻松地把解法翻译成代码。"[^2] 他列的可接受备选是 Go、Ruby、PHP、C#、Swift、Kotlin。要*避开*的、因为面试官读不顺的，是 Haskell、Erlang、Perl、C、MATLAB。[^2]

Python 在三个具体维度上胜出。一道滑动窗口中档题，Python 写 15 到 25 行，Java 要 30 到 50 行。标准库开箱就覆盖 `Counter`、`deque`、`heapq`、`bisect`、`itertools` 和 `lru_cache`。白板也宽容：不用写类型声明，不用套类壳子，切片语法能把逻辑压成一行。

Python 只在"陷阱题"意义上躲开了整数溢出——它压根不会溢出，因为按 PEP 237，`int` 是任意精度。[^3] Java、C++ 和 Go 都得交这笔溢出税。凭据在[面试数学](./04-math-for-interviews.md)和[语言惯用法](./05-language-idioms.md)里。

该选别的语言的情形：

- **如果你工作中已经能流畅写某门语言，就接着用它。** Yangshun 说得很直接："如果你在工作中用 Java 已经有一段时间，又没时间去把另一门语言练到顺手，我建议就守住 Java，而不是为了面试从零捡起 Python。"[^2] 切换成本是真实的，而且通常大于收益。
- **C++ 用于交易公司、字节跳动核心系统、Bloomberg。** 在最难的题上 TLE 风险最低，也是这些地方的文化默认。
- **Java 用于任何大型 JVM 技术栈。** Amazon、Bloomberg 后端、微软 Azure 的部分团队。Java 人在 Java 公司这个信号，是 interviewing.io 在数据里找到的唯一语言效应。[^1]
- **Go 用于 Stripe、CNCF 系基础设施公司，以及 Cloudflare 和字节跳动基础设施的后端岗**，前提是你的目标不是 Google（见下一个问题）。

其余几乎一切情况：Python。

## 我的语言在目标公司能用吗

能用，只有一家例外。Yangshun 的 *Tech Interview Handbook* 记录了业内唯一一条硬性限制："大多数公司允许你用任意语言写代码，我知道的唯一例外是 Google，他们的算法编码面试只允许候选人从 Java、C++、JavaScript 或 Python 里选。"[^2] Go 在 Google 的招聘页面上是允许的（他们内部就在用），但不在算法编码轮的可选清单里。

其他大厂的默认值都是软信号。它们说明的是这家公司的资深工程师会默默注意到什么，而不是他们要求什么。下面这些按公司列的指引，多数来自候选人的反馈和招聘官的记录，由本手册的面试调研汇编整理，而不是公司官方文档。[^4]

| 公司 | 候选人反馈 | 硬性限制 |
|---|---|---|
| Google | 面试中最常见的是 Python；Java 和 C++ 广泛接受 | **有：**仅限 Java、C++、JavaScript 或 Python[^2] |
| Meta | SWE-Product 用 Python，基础设施用 C++，前端用 JavaScript；面试接受这四门中的任何一门[^4] | 算法轮无限制 |
| Amazon | 后端长期惯例是 Java，但 Python 完全接受；AWS 系统用 C++[^4] | 无 |
| Microsoft | 许多团队内部默认 C#；Java 和 Python 同样受欢迎[^4] | 无 |
| Apple | 看团队：iOS 期望 Swift 或 Objective-C；基础设施接受 Python、Java、C++、Go；ML 期望 Python[^4] | 按团队而定；问招聘官 |
| Netflix | 后端与数据以 Python 为主；遗留服务里有 Java[^4] | 未听说有 |
| Stripe | 明文政策是与语言无关；实际上候选人用 Ruby、Python、Go[^4] | 无 |
| ByteDance / TikTok | 核心系统用 C++，基础设施用 Go，ML 用 Python[^4] | 未听说有 |
| Bloomberg | 历史上的默认是 C++（终端那套代码库）；Python 和 Java 也行[^4] | 无 |

唯一值得背下来的一条：如果你目标是 Google 而你又选了 Go，要么换语言，要么换目标。其他每一行都只是软偏好，不是墙。

## 校招与资深候选人的语言信号有何不同

对校招候选人，唯一重要的信号是*任意一门语言的流利度*。interviewing.io 的数据毫不含糊：挂掉的人不是那些选了 Java 而不是 Python 的人，而是那些选了自己并不流利的语言的人。[^1] 校招的信号是"我能在时间压力下用这门语言写出流畅的代码"，而 Python 用最短的练习时间就能满足这个信号。

对资深候选人（L5+ / E5+），标定会变。一位资深工程师在重度 Java 的公司写 Python，不会因此挂掉整轮，但面试官会记下：这人懒得去了解公司的技术栈。interviewing.io 发现的"Java 人在 Java 公司"效应，对一般候选人在统计上显著；在资深层级，惯用法的期望更严，效应量很可能更大。[^1] 如果你只盯一家主目标公司，就用他们的默认语言面试，哪怕你日常写的是别的。代价：40 到 80 小时上手。收益：一个"我已经知道你们在用什么"的信号。

对 staff 和 principal 候选人，轮次结构变了，语言选择淡出。Stripe 的 staff 轮是一页纸文档加一场演讲。Apple 的资深轮包含在候选人最强领域的深入追问。Meta 的 E7+ 候选人会遇到 AI 辅助的多文件轮，这一轮里日常工作的语言按定义就是赢家。你写得最好的是什么，就写什么。

## 什么时候换语言才算太晚

备赛中途换语言，成本随已投入小时数超线性增长。下面的数字来自重新内化[语言惯用法](./05-language-idioms.md)里八个跨语言惯用法单元（带比较器排序、哈希表字面量、deque、堆、递归深度上限、泛型、相等与哈希、切片）所需的时间，每单元约 30 到 90 分钟，再乘上 200 到 400 道题。

| 已练习小时数 | 切换成本 | 建议 |
|---:|---|---|
| 0 到 5 | 可忽略 | 随便换。成本就是一个周末。 |
| 5 到 20 | 10 到 20 小时 | 有真实理由才换：目标公司的硬性限制，或者你确实更熟的另一门语言。 |
| 20 到 50 | 30 到 60 小时 | 只在硬阻塞时换：目标变成 Google 而你选了 Go，或者某块缺失的标准库一直咬你。 |
| 50 到 200 | 60 到 150 小时 | 别换。绕开这些限制。 |
| 200+ | 等于从零开始 | 任何情况下都别换。用手上的东西交货。 |

操作规则很短：**头五小时做决定，只在头二十小时内换。** 过了这个点，换语言的成本已经高于绕开那个让你想换的原因的成本。

换语言的真理由：你的目标从一家对 Python 友好的公司变成了 Google，而你选了 Go；你已经练了 30 小时，而 Python 的递归上限和字典迭代顺序语义在图论题上一直咬你；你三个月后要加入一个 Java 团队，想让面试准备同时充当入职上手。

换语言的烂理由："C++ 看起来更厉害"（interviewing.io 的数据不支持这一点；各语言通过率在统计上相同[^1]）；"公司内部用 Rust"（Rust 不是常见的面试语言，用面试官读不懂的东西是自废武功[^2]）；"我听说 Python 写起来快"（平均而言没错，但如果你已经写了 30 小时 Java，写作速度的收益会被重新学习的成本吃掉）。

## 有哪些坑要当心

一张速查卡。每行都是一个真实能让面试翻车的 bug，按语言分列。细节在[语言惯用法](./05-language-idioms.md)里；这里给一眼能看完的版本。

| 坑 | Python | Java | C++ | Go |
|---|---|---|---|---|
| 整数溢出 | 无（PEP 237）[^3] | 有（`int` 是有符号 32 位，最大 `2^31 - 1`） | 有（有符号溢出是 UB） | 有（用 `int64` 或加保护） |
| 默认递归深度 | 1,000（在模块顶部调到 `10**6`） | 约 5,000 到 20,000（JVM 栈约 1 MB） | 约 50,000 到 200,000（Linux/macOS 上 8 MB 栈） | 无上限（goroutine 栈自动增长） |
| 哈希表迭代顺序 | 插入顺序（3.7+） | `HashMap` 中未定义；用 `LinkedHashMap` | `unordered_map` 中未定义 | 未定义，且设计上随机化 |
| 堆的默认极性 | 最小堆（`heapq`） | 最小堆（`PriorityQueue`） | **最大堆**（`std::priority_queue`） | 取决于 `Less` |
| 容器头部弹出 | `list.pop(0)` 是 O(n)；用 `collections.deque` | `LinkedList` 可用；`ArrayDeque` 更快 | `std::deque` 是标准做法 | `slice[1:]` 是 O(n)，而且共享底层数组 |
| 排序稳定性的默认值 | 稳定（TimSort） | `Arrays.sort(Object[])` 稳定；`int[]` 不稳定 | 不稳定；用 `std::stable_sort` | 不稳定；用 `slices.SortStableFunc` |
| 比较器溢出 | 无 | 有（`a - b` 违反全序；用 `Integer.compare`） | 有（溢出的减法行为是 UB） | 有（用 `cmp.Compare`） |

"我本地跑通了但面试挂了"的故事，大半落在其中两行。递归深度那行咬你，是因为一个在 `n = 100` 上跑通的 Python 解法，到 `n = 1,000` 就炸了——它越过了 1,000 帧的默认上限；在文件顶部加 `sys.setrecursionlimit(10**6)` 是一次性修法。整数溢出那行咬你，是因为 `a * b` 在 Java、C++ 或 Go 里悄悄回绕了；修法是在做乘法*之前*拓宽到 `long` / `long long` / `int64`，而不是之后。

## 本周做什么

如果你手上没有已用顺的语言：把这章再打开一遍，用 Python 写 `count_freqs("abracadabra")`（用 `Counter` 就两行），然后从[计算复杂度与 Big-O](./01-complexity-big-o.md)开始。沉没成本目前是零。在跨过五小时之前做决定。

如果你已经有一门用顺的语言：守住它。你学一门新语言要花的 40 到 80 小时，收不回来——除非你手上这门在 Yangshun 的避让清单里（Haskell、Erlang、Perl、C、MATLAB），或者你唯一的目标是 Google 而你目前在写 Go。

[语言惯用法](./05-language-idioms.md)是你以后会反复回查的跨语言参考。[第 14 部分 — 面试框架](../../../dsa/part-14-interview-framework/00-pattern-recognition.md)是更宏观的备赛心智模型。语言已经定了。接下来 100 到 400 小时的练习，才是真正决定成败的部分。

## 参考文献

[^1]: Lerner, A. and Holtz, D., "We analyzed thousands of technical interviews on everything from language to code style. Here's what we found," interviewing.io blog, [interviewing.io/blog/we-analyzed-thousands-of-technical-interviews-on-everything-from-language-to-code-style-here-s-what-we-found](https://interviewing.io/blog/we-analyzed-thousands-of-technical-interviews-on-everything-from-language-to-code-style-here-s-what-we-found).

[^2]: Tay, Y., "Which programming language to use for coding interviews," *Tech Interview Handbook*, [techinterviewhandbook.org/programming-languages-for-coding-interviews](https://www.techinterviewhandbook.org/programming-languages-for-coding-interviews/).

[^3]: Zadka, M. and van Rossum, G., "PEP 237 — Unifying Long Integers and Integers," [peps.python.org/pep-0237](https://peps.python.org/pep-0237/).

[^4]: Per-company defaults synthesized from candidate reports and recruiter conversations on interviewing.io's company-specific guides (refreshed Q4 2025 / Q1 2026) and the krishnadey30/LeetCode-Questions-CompanyWise community tags.
