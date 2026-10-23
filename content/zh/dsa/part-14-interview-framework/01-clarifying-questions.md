---
title: "头五分钟：澄清问题"
description: "头五分钟该问什么。面试官真正在打分的那些问题、能体现资深度的提问方式，以及让最多候选人输掉整轮的那一种失败模式。"
slug: 01-clarifying-questions
part: 14
chapter: 1
difficulty: intermediate
prerequisites:
  - part-14-interview-framework/00-pattern-recognition
date_updated: 2026-05-25
---

# 头五分钟：澄清问题

面试官念出题目。*“Reverse a string.”* 三个词。共享屏幕那头安静下来。你有两秒钟，之后这份沉默就开始像是答错了，而候选人此刻最常做的事，恰恰是最容易输掉整轮的那一步：开始写代码。

在 interviewing.io 平台上 600 场有记录的技术面试中（2023），Ian Douglas 把“不问澄清问题”列为候选人最常见的五种失败模式之一。[^1] Codeintuition 对一场 45 分钟 Google 面试轮的拆解（2026）从另一个方向给同一个观察配上了数字：45 分钟里有 3 到 5 分钟花在澄清上，而真正被记为正面信号的是**约束类问题**，不是“这是什么意思？”这类问题。[^2] 前者收紧规格，后者只告诉面试官你没仔细读题。

本章讲该问什么、按什么顺序问、什么时候停。

## 先问什么？

开口之前，先用自己的话把题目复述一遍。三十秒，不能再多。从面试官刚说的话里挑一个具体的点，抛回去：*“所以我们有一个整数数组，想找出两个元素的下标，使它们的和等于目标值。输出是一对下标，不是那两个值本身。对吗？”* 这一步能早点抓到误读，给面试官纠正你的余地，也把“我在想”的沉默变成一次契约谈判。

然后按这个顺序问：输入形态、输出契约、约束、边界情况。多数题目的缺口都在前两项，其余的顺着就出来了。

**输入形态。** 类型、范围、有无符号、可变性、是否有序。*“输入是数组、流，还是生成器？”* 流式输入会逼出单趟算法。*“值可以为负吗？”* 负数会破坏教科书双指针解法所依赖的滑动窗口单调性。*“我可以原地修改输入吗？”* 这决定了是 O(1) 额外空间还是 O(n)。

**输出契约。** 返回类型、顺序、无解时的行为。*“如果不存在答案，我返回什么：`null`、`-1`，还是抛异常？”* LeetCode 1 的契约是 `-1`；生产代码会去用 `Optional`。同一道题，不同的签名，假设错了就会挂在隐藏用例上。

**约束。** `n` 的规模、时间与空间目标。*“`n` 最大到 10^5 还是 10^9？”* 答案直接决定算法类别。10^9 在你写下第一行之前就已经排除了任何平方级算法。

**边界情况。** 空输入、单元素、全部相同、边界值。Codeintuition 标记为强的那位候选人（2026）在一道岛屿数量题上主动覆盖了空网格的情况。[^2] 旁边那位没有，这就是为什么一个进了下一轮。

上面每一个问题，其**答案都会改变代码**。这就是检验标准。如果答案不改变代码，这个问题就不配占那个位置。

## 哪些问题才真正重要？

拿个真实例子。*“给定一个整数数组，找出两个数，其和等于目标值。”* 这就是 [LC 1 — Two Sum](https://leetcode.com/problems/two-sum/)，只是表述含糊。Ian Douglas 特意用一道几乎一样的热身题，因为大多数候选人听到“数组”就不去核实了；他那道题说的是“grouping”，他在看对方会不会问出来。[^1]

有五个问题值得问。*“数组有序吗？”* 有序就能用双指针，O(1) 额外空间；无序就只能上哈希表。*“同一个下标能用两次吗？”* 这影响循环内层的检查。*“有重复元素吗？”* 当存在两组合法解时，重复会改变返回契约。*“没有匹配时返回什么？”* 答案决定函数签名。*“允许负数吗？”* 这能在“不能低于零”那套论证咬到你之前就把它排除掉。

五个问题，九十秒，你就把题目收敛到一个算法上。这正是面试官在打的分。

反面例子：一个候选人问*“数组是零下标起吗？”*或者*“循环变量该叫 `i` 吗？”*，那是在打勾，不是在收窄规格。一篇拆解 45 分钟 FAANG 面试轮的 substack 文章（Rafay Abbasi，2026）直接点出了这一点：*“问那些答案不会改变代码的问题”*是一种有据可查的失败模式。[^3]

## 预算多少？

三到五分钟。不是正好五分钟。这是上限，不是目标。

| 子阶段 | 时间 | 发生了什么 |
|---|---|---|
| 复述题目 | 约 30 秒 | 用自己的话复述；暴露误读 |
| 输入形态 | 约 60 至 90 秒 | 类型、范围、可变性 |
| 输出契约 | 约 30 至 60 秒 | 返回类型、顺序、无解时的取值 |
| 约束 | 约 30 至 60 秒 | `n`、时间/空间目标 |
| 边界情况 | 约 30 至 60 秒 | 空、单元素、并列 |
| 确认方向 | 约 30 秒 | 复述你选的方向；让面试官纠正 |

一道全新的中等题这样跑下来，顶到大约四分钟。[^2][^3] 《Cracking the Coding Interview》里的“Listen / Debug example / Special cases”序列（McDowell，第 6 版，2015）把同样这三步压进同一个时间窗。[^4] AlgoCademy 公开的指导也落在 2 到 5 分钟这个区间。[^5]

不属于这个预算的：设计算法、在两个解法之间权衡、或者问“我该讲多细？”这类流程性的元问题。那些都在后面。花 15 秒对齐沟通节奏（“我先想大约一分钟，然后讲思路，再写代码”）没问题；花好几分钟聊你平时怎么工作就不行。

真遇到一道热身级的简单题（[LC 344 — Reverse String](https://leetcode.com/problems/reverse-string/)），预算压到总共 60 到 90 秒。为*“反转一个字符串”*花四分钟澄清，和不澄清是同一种失败模式。

## 什么时候问得太多反而成了失败？

已经过去六到八分钟，你在问第十二个问题。面试官会读成拖延。

中级候选人把“要问澄清问题”当成一个数量目标。他们照着清单往下走（输入、输出、约束、边界、分布、运行环境），却不去想某个答案是否会改动一行代码。那篇 substack 文章逐字引用了面试官的记录：*“把澄清当成打勾。重点不是问问题，而是收窄问题。每个问题都应该排除一种可能的解读。”*[^3]

同一个阶段里，还有三种动作会让候选人输掉整轮。

**问完就不听答案。** 你问 `k` 是否总是合法。面试官说是。然后你防御性地写下 `if (k <= 0) return [];`，现在面试官觉得你要么没在听，要么不信任你刚谈成的契约。把那个守卫去掉。[^3]

**把问解法伪装成提问。** *“我该用哈希表还是先排序？”* 听起来像在钓答案。带着理由说出你的选择，再请求确认：*“我打算用哈希表，做到 O(n) 查找，代价是 O(n) 空间。这样合理吗？”* 同样的答案，不同的信号。[^5]

**含糊的复述，掩盖了误读。** *“所以我们要找最常见的那些东西”* 完全没告诉面试官你到底懂没懂，而你接下来要去写一道略微错的题。用题目里的具体元素复述：输入类型、输出类型、那个能消歧的约束。

这三种情况的解法是同一个。把澄清当成**两个工程师之间的契约谈判**，而不是面试官在考你。

## 什么时候该少问？

三种情况会压缩预算。

**题目自带书面规格。** 两个示例、明确的约束（`1 <= n <= 10^5`）、值域。剩下要问的只是并列时的取舍和一两个边界情况。别去发明已经被填上的缺口。

**面试中途的追问。** *“现在如果网格会环绕呢？”* 讨论范围已经确定了。一两个增量澄清，然后写代码。

**热身级的简单题。** 反转字符串。有效括号。面试官在衡量你能不能快速推进。总共六十到九十秒：类型、原地还是返回新串、ASCII 还是 Unicode，结束。

这三种之外，每次都要跑完整的澄清阶段。本章标题说*五分钟*，因为那是八份被调研资料共同收敛到的上限；下限更接近两分钟。

## 本周做什么

从你的题单里抽两道题，一道简单一道中等，在解题之前把澄清阶段出声跑一遍，掐着表。中等题三分钟，简单题九十秒。录下来。回听时挑出那些答案本来不会改变代码的问题，下次把它们划掉。

两道示例题，想一想即可，不必在本章里解：

- [LC 1 — Two Sum](https://leetcode.com/problems/two-sum/) [简单]。表层歧义（是否有序、返回格式、无解行为）是练输入形态这条线最干净的素材。
- [LC 146 — LRU Cache](https://leetcode.com/problems/lru-cache/) [中等]。题目是*“设计并实现”*，这就把 API 契约逼到台面上。`get` 在未命中时返回什么？缓存是线程安全的吗？复杂度目标是 O(1) 最坏情况还是均摊？答案决定数据结构。

这两题在各自的算法章节里都是核心（CORE）题；本章只是引用它们。澄清这项技能本身，是靠在你正在练的题目上反复钻出来的。

[模式识别：见效最快的那项技能](./00-pattern-recognition.md) 在本章上游。输入形态一旦钉住，模式识别就会在输入输出签名上触发，告诉你该抓哪个算法。整轮面试余下时间的沟通，是后面那一章。

## 参考文献

[^1]: Douglas, I., “I've conducted over 600 technical interviews on interviewing.io. Here are 5 common problem areas I've seen,” interviewing.io blog, 1 May 2023, [interviewing.io/blog/ive-conducted-over-600-technical-interviews-on-interviewing-io-here-are-5-common-problem-areas-ive-seen](https://www.interviewing.io/blog/ive-conducted-over-600-technical-interviews-on-interviewing-io-here-are-5-common-problem-areas-ive-seen).

[^2]: Codeintuition, “Google Coding Interview Experience: 45 Minutes Decoded,” accessed 21 March 2026, [codeintuition.io/blogs/google-coding-interview-experience](https://www.codeintuition.io/blogs/google-coding-interview-experience).

[^3]: Abbasi, R., “The Anatomy of a Perfect FAANG Coding Interview (A 45-Minute Breakdown),” DG Learning Substack, 13 April 2026, [dglearning.substack.com/p/the-anatomy-of-a-perfect-faang-coding](https://dglearning.substack.com/p/the-anatomy-of-a-perfect-faang-coding).

[^4]: McDowell, G.L., *Cracking the Coding Interview: 189 Programming Questions and Solutions*, 6th ed., CareerCup, 2015, ISBN 978-0-9847828-5-7.

[^5]: AlgoCademy, “Should You Ask Clarifying Questions Before Coding in Interviews?” AlgoCademy Blog, accessed 20 May 2026, [algocademy.com/blog/should-you-ask-clarifying-questions-before-coding-in-interviews](https://algocademy.com/blog/should-you-ask-clarifying-questions-before-coding-in-interviews/).
