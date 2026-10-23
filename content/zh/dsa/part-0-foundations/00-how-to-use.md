---
title: "如何使用本手册"
description: "怎么读这本手册：写给谁、哪些可以跳过、怎么选语言，以及每份 DSA 资料都躲着不答的那四个问题。"
slug: 00-how-to-use
part: 0
chapter: 0
difficulty: beginner
prerequisites: []
date_updated: 2026-05-24
languages: [python, java, cpp, go]
ladder:
  core: []
  stretch: []
---

# 如何使用本手册

大多数读者是在试过两三个别的 DSA 资源、最后什么计划也没带走之后才翻到这一页的。Blind 75 给了你 75 道题，却没给你一条主线。NeetCode 150 把题按类别分好组，然后让你自己去刷。Striver 的 A2Z 清单给了你 474 道题，然后眼看着三分之一的人在第六周退出。它们都没回答你真正带着来的那四个问题：这件事值不值得我花时间、我从哪儿开始、我怎么知道自己做完了、以及我用哪门语言写。

这一页回答这四个问题。下一章从你合上这一页的地方开始。

## 你将看到什么

本手册共十五个部分、约 120 章。第 0 部分——也就是你现在所在的位置——讲的是每个算法章节都默认你会的那七项基本功：复杂度记号、递归、位运算技巧、面试里真会出现的数学，以及让 Python、Java、C++、Go 读起来顺溜的语言习惯用法。第 1 到第 13 部分是算法本身，按数据结构家族和模式组织：线性结构、查找/排序、双指针与滑动窗口、栈、链表、树、递归与回溯、图、DP、贪心、位运算、字符串，以及"设计数据结构"类题目。第 14 部分讲面试流程本身：模式识别、澄清问题、沟通、模拟面试演练、Amazon LP 叙述，以及 Meta 的 AI 辅助编码轮。

与这些部分并行的还有两块配套内容。模式库（Patterns Library）是 37 篇短文，围绕"什么时候该用 X 而不是 Y"这个问题展开，适合在你见过底层算法之后横向阅读。附录 C 收录了五十篇深度题解，每个被章节标上 ★ 的招牌题都有一篇。每篇题解都是完整的 4,000 到 6,000 字：澄清问题、三到五种从朴素到最优的解法推演、边界情况、取自 LeetCode Discuss 的常见 bug、四语言参考实现，以及逐步跟踪。

全书共有 92 个手工制作的交互式组件，用来把光靠文字讲不清的算法动起来。其中四十二个在章节正文里，另外五十个在附录 C 的题解里。

## 三条阅读路径

本手册按三种明确的路径设计体量。开始之前先选一条。

| 路径 | 题目数 | 用时 | 适合谁 |
|---|---|---|---|
| **推荐** | 160（进阶路线 Progression Tracks 上的 CORE 题） | 约 107 小时 / 约 3 个月 | 默认选择。为 12 周冲刺准备 FAANG 级面试循环的读者。比 NeetCode 150 大；[^1] 比 Striver A2Z 小得多。[^2] |
| **核心掌握** | 约 275（每个算法章节里的全部 CORE 题） | 约 5 个月 | 想在全部 15 个部分都拿到章节掌握确认的读者。 |
| **全面** | 约 445（所有章节的 CORE + STRETCH）加 50 篇题解 | 约 7 到 8 个月 | 自定进度的学期制。像用教科书那样用这本手册。 |

这些数字是对着几个显而易见的基准校准出来的。Blind 75[^3] 在广度上太小；NeetCode 150[^1] 是当下 FAANG 的基线；Striver A2Z[^2] 则是自学完成率崩掉的地方所在。推荐路径定在 160 题，是为了让刻意练习在三个月内保持可控，同时不掉进 Striver 那种体量的课程所面临的悬崖。[^4]

如果你分不清哪条路径适合自己，选推荐。路径大小是对投入的预估，不是承诺。以后你可以把推荐路径扩展成核心掌握——把没做完的章节的 STRETCH 梯度补完就行。

## 章节如何组织

每个算法章节都以一个**只给链接的 LeetCode 梯度**收尾，三到五道题，分成两档。

CORE 是三道题，横跨 Easy、Medium、Hard 三档。三道全部解出，章节掌握即被点亮。STRETCH 是一到两道可选的延伸题。每章有且只有一道 CORE 题带 ★，也就是本章的招牌题。★ 题同时也会得到本章的实例推演、本章的动画组件（如果存在），以及附录 C 的一篇题解。

梯度条目刻意保持最小：LC 编号、题名、难度括号、单一模式标签、链接。没有公司标签（那些在 `content/dsa/per-company/` 的分公司路线里）、没有时间估计、没有内联提示、没有代码。题目陈述和代码运行都由 LeetCode 负责；本手册的价值在于**排序与分组**。教学内容在章节正文、实例推演、记忆卡片和附录 C 的题解里。

大约二十一章是基础或框架内容：第 0 部分的元章节、第 14 部分全部，以及另外几章。它们不受 Easy/Medium/Hard 跨度要求约束，配零到两道题，不要求覆盖各档难度。这一页就是其中之一。这里没有梯度，因为你还没见过任何算法。

## 组件如何工作

约 45% 的章节带一个动画组件。各处的约定都一样。

每个组件默认**停在第 0 步**。没有任何自动播放。你可以前进一步（Right）、后退一步（Left）、重置（R），或者播放与暂停（Space）。每个组件都提供一个速度选择器、一份静态关键帧回退（在 JavaScript 关闭或设置了 `prefers-reduced-motion: reduce` 时使用），以及一行 `aria-live` 状态提示，用平实的英文播报当前这一步。[^5]

视觉语言是固定的。排序和图组件使用 Wong 的八色调色板，[^6] 选它的目的是让每一种状态变化在红色盲、绿色盲和蓝黄色盲下都能区分开。活动状态是橙色。已访问的格子是天蓝色。被接受的答案是蓝绿色。被拒绝是朱红色。全书没有任何地方用赤裸的红/绿配对来表达"好与坏"或"已访问与未访问"，因为大约 8% 的男性读者和 0.5% 的女性读者是红绿色盲。[^6]

组件复杂度按 Bostock 2014 年的三级约定来定。[^7] 大多数章节拿到一张 Level-0 主图加一个 Level-1 过程组件。Level 2——完整内部状态可见——只留给全书最难的四个算法：AVL 旋转、Dijkstra、KMP 的失败函数，以及带懒惰传播的线段树。理由就是 Bostock 自己的说法：白盒可视化最有可能把问题讲清楚，同时对读者的负担也最重。我们只在算法值得的地方付这份负担，绝不提前付。

默认暂停这个立场比 WCAG 2.2 SC 2.2.2 的要求更严格（后者要到第五秒才生效）[^8]，也与 Mayer 的分段原则一致：对新手来说，学习者自定节奏优于系统定节奏，效应量 d = 0.70。[^9]

## 四语言策略

书中每个代码块都渲染成带 **Python、Java、C++、Go** 标签页的组件。这个选择是刻意的，不是历史遗留。

Python 是基准文本。正文里说"数组 `nums`"时，这个名字就出现在 Python 标签页里。Python 标签页的迭代模型（`enumerate`、`zip`、`collections.Counter`、`heapq`）是四者中最接近算法伪代码风格的，而且 Python 是 LeetCode 的默认提交语言。

Java 存在，是因为在大多数大型工程组织里，Java 21 仍是后端面试的首选语言：Amazon、Bloomberg、Stripe 的 JVM 技术栈、Google 的部分团队。Java 标签页还会暴露装箱开销（`PriorityQueue<Integer>` 的每次 `offer` 和 `poll` 都会自动装箱）和溢出陷阱（用 `Integer.compare(a, b)` 而不是 `a - b`），这些是更高级的语言替你藏起来的。

C++ 是系统和量化面试的必需品。基线是 C++17。有符号溢出是未定义行为。编译过程毫不宽容。这两点都有教学价值。

Go 是必需品，因为它已经成为现代基础设施型公司（Stripe、Cloudflare、ByteDance）第二常见的后端语言。Go 标签页还会在每个涉及子数组视图的章节里暴露切片别名这个坑——这是全书章节代码最常见的 bug 来源。

同一算法、可能不同的数据结构，是允许的，而且往往必需。LRU 缓存在 Python 里是 `OrderedDict`，Java 里是 `LinkedHashMap`，C++ 里是 `list<pair> + unordered_map<key, iterator>`，Go 里是 `container/list + map`。四种都正确。

面试只用一门语言。读其他语言的版本，是为了加深模式识别。[选择你的面试语言](./06-choosing-your-language.md) 是决策矩阵；[语言习惯用法](./05-language-idioms.md) 是你会反复回来查的逐项对照参考。

## 如何阅读

三条规则，按优先级排列。

**不要跳过第 0 部分。** 每个算法章节都会引用它。大多数读者想跳；大多数读者不该跳。Skiena 在 *The Algorithm Design Manual* 里讲过同样的道理：那本书有一半篇幅在讲技巧，正是因为跳过基础就意味着每道题都要重新推导一遍。

**按部分顺序读，不要按主题顺序读。** 第 1 到第 13 部分建立在第 0 部分之上，彼此之间也互相依赖。模式库是一块横向的内容。读完一章再去看它，而不是之前，因为每个模式页都是一个"决策问题"，只有在你见过底层算法之后才说得通。

**继续往下之前先把 CORE 题做完。** 从头解题胜过反复重读。测验效应（testing effect）——提取练习比重新学习带来更强的长期记忆，即使没有反馈也成立——在教育心理学研究里已经立了四十年，也是 CORE 梯度设计和章节记忆卡片共同的底层依据。[^10]

## 这不是什么

它不是竞赛编程书。Codeforces 的题和面试题不一样，头 50 小时之后技巧路线就分岔了。如果你要 CP，去读 Halim 的 *Competitive Programming Handbook*。

它不是计算机科学教科书。证明并不完备。那些偏门的复杂度类别（平滑分析、竞争比、多项式层级）不在范围内。CLRS[^11] 和 Erickson 才是对应的参考。

它不是速查表。确实有个模式库把模式决策压缩了，但真正在教学的是章节本身。只翻不做的读者，在面试压力下做不完一道 Medium。

它是一本工作手册。你读一章，解掉它的三道 CORE，然后往下走。记忆卡片按间隔重复重新加载。你停止拖动时，组件就暂停。你一旦决定开始，这本书就让开路。

接下来是[计算复杂度与 Big-O](./01-complexity-big-o.md)。今天晚上就读它。

## 参考文献

[^1]: NeetCode 150 problem list, [neetcode.io/practice?tab=neetcode150](https://neetcode.io/practice?tab=neetcode150).

[^2]: takeuforward (Striver), "Strivers A2Z DSA Course/Sheet", [takeuforward.org/dsa/strivers-a2z-sheet-learn-dsa-a-to-z](https://takeuforward.org/dsa/strivers-a2z-sheet-learn-dsa-a-to-z).

[^3]: Yangshun Tay, "Blind 75" curated LeetCode list, originally posted on the Blind platform 2018, [techinterviewhandbook.org/grind75/](https://www.techinterviewhandbook.org/grind75/)

[^4]: Ericsson, K.A., Krampe, R.T., Tesch-Römer, C., "The Role of Deliberate Practice in the Acquisition of Expert Performance", *Psychological Review*, 100(3), 363-406, 1993, [eric.ed.gov/?id=EJ471947](https://eric.ed.gov/?id=EJ471947).

[^5]: W3C, "Understanding Success Criterion 2.2.2: Pause, Stop, Hide" (Level A), WCAG 2.2 Understanding Docs, [w3.org/WAI/WCAG22/Understanding/pause-stop-hide](https://www.w3.org/WAI/WCAG22/Understanding/pause-stop-hide).

[^6]: Wong, B., "Points of view: Color blindness", *Nature Methods*, 8, 441 (2011), DOI 10.1038/nmeth.1618, [nature.com/articles/nmeth.1618](https://www.nature.com/articles/nmeth.1618).

[^7]: Bostock, M., "Visualizing Algorithms" (June 26, 2014), [bost.ocks.org/mike/algorithms](https://bost.ocks.org/mike/algorithms/).

[^8]: MDN Web Docs, `prefers-reduced-motion`, [developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion).

[^9]: Mayer, R.E. and Pilegard, C., "Principles for Managing Essential Processing in Multimedia Learning", in Mayer (ed.) *The Cambridge Handbook of Multimedia Learning*, 2nd ed., Cambridge University Press, 2014.

[^10]: Roediger, H.L. and Karpicke, J.D., "Test-Enhanced Learning", *Psychological Science*, 17(3), 249-255, 2006, [pubmed.ncbi.nlm.nih.gov/16507066](https://pubmed.ncbi.nlm.nih.gov/16507066/).

[^11]: Cormen, T.H., Leiserson, C.E., Rivest, R.L., Stein, C., *Introduction to Algorithms*, 4th ed., MIT Press, 2022, ISBN 978-0-262-04630-5.
