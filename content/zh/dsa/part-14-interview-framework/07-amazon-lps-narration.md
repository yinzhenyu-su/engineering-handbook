---
title: "Amazon 领导力准则的讲述"
description: "面向技术轮的 STAR 格式微型叙述，以及行为轮和 Bar Raiser 轮期待候选人带着进场的八到十二个完整故事。"
slug: 07-amazon-lps-narration
part: 14
chapter: 7
difficulty: intermediate
prerequisites:
  - part-14-interview-framework/06-amazon-lps-brief
date_updated: 2026-05-25
---

# Amazon 领导力准则的讲述

顶级公司里，只有 Amazon 是光靠算法熟练度过不了面试 loop 的。每一轮，包括那两轮编码面，都会在同一份笔记里给领导力准则（Leadership Principles，LP）打分——技术信号也记在这份笔记上。代码很强、行为面全程沉默的候选人，从 debrief 里拿到的结论都一样：两道题都做出来了，每个澄清问题都用一句话答了，最后因为信号不足被判 no-hire。要补的范围其实不大：编码轮准备十来个练熟的微型叙述，行为轮和 Bar Raiser 轮准备八到十二个完整故事。

准则本身（经典的 16 条、从 14 条到 16 条的演变、哪五条在编码轮出现最多）请先读 [Amazon 领导力准则入门](./06-amazon-lps-brief.md)。下面这一页假设你已经知道它们各自讲什么。

## 每个故事应该讲多久

说出口大约四分钟。Amazon 官方发布的面试指南让候选人用 STAR，并且要讲清楚自己个人的贡献；一个 STAR 回答内部通行的节奏是：Situation 一分钟、Task 一分钟、Action 九十秒、Result 三十到六十秒。[^1][^2] 故事超过五分钟，Bar Raiser 会打断。不到两分钟，他们会追问。

四个部分的时长要刻意分配。一个故事要是这么开场——“这个项目是把我们的支付栈从 Postgres 迁到 Spanner，做了六个月……”——光 Situation 就烧掉九十秒，剩下两分钟得装下所有真正重要的东西。Situation 用一句话加一个数字带过。*“2024 年第三季度，我们的下单服务在压力下会丢 0.3% 的购物车。这块是我负责的。”* 这样面试官才跟得上。

## STAR 是什么，必须要用吗

STAR 是 **Situation、Task、Action、Result**（情境、任务、行动、结果）。这个缩写是 Development Dimensions International（DDI）在 1970 年代推广开的，属于他们 *Targeted Selection* 方法论的一部分；它和 Tom Janz 的学术方法 Behavior Description Interviewing（1986）同源[^3]，共同的源头是 John Flanagan 的关键事件法（Critical Incident Technique，1954）。[^4] 历史你不用记。结构必须记，因为 Amazon 的面试指南明确点名 STAR，还说不用它的回答往往会漏掉“谁做的”和“结果是什么”。[^2]

它不是拿来背的台词。面试官不是在打分“候选人有没有把四个字母都念到”。他们打的是：你讲的是不是*具体*那件事（不是一类事），*你本人*负责什么，*你本人*做了什么，以及因此发生了什么变化。STAR 就是一张清单，专门兜住 Bar Raiser 在 debrief 里最爱挑的四种缺失：情境含糊、归属不清、没有具体动作、没有可量化的结果。[^2]

准备 Amazon 面试的人，最该改的一处是代词。故事一滑进“我们”，分数就塌。Amazon 的指南说得很直接：凡是你本人推动的动作，把“我们”改成“我”。[^2] *“我们把一套功能开关系统推上线了”* 改成 *“这套功能开关系统是我设计的。两个工程师帮我实现，上线是我负责的。”* 同一个项目，两个故事。

## 后端故事里怎么体现 Customer Obsession（客户痴迷）

多数候选人以为 Customer Obsession 得有个面向客户的产品界面。不是这样。这条准则的官方原文是“领导者从客户出发，倒推工作”，还有“全力以赴赢得并保持客户的信任”。[^5] 客户可以是内部的。你的客户可以是凌晨三点被呼机叫起来的 on-call 工程师，也可以是查询打到你这个服务上的数据科学家。

一个后端版的 Customer Obsession 故事，有三块承重结构。客户是谁，要具体点名。他们需要什么，要站在他们的目标上说，不是站在你的工作量上说。你为了满足它改动了系统的什么，带上指标。

*“客户是我们内部的机器学习团队。卡他们的是特征存储的一次查询，p99 要 800 毫秒，逼得他们只能每天晚上预计算特征，推上线的预测都是过期的。我推动了表结构改动，加上覆盖索引。p99 降到 40 毫秒。下一个季度他们就切到实时推理了。”*

这个故事能过 Customer Obsession，因为它一开口讲的是客户的目标，不是你的代码。它同时也过了 Deliver Results（交付结果，因为有指标），还轻轻碰到 Dive Deep（深入细节，因为改到了表结构这一层）。一个故事，三条准则。你要的就是这个形状。

## 需要准备多少个故事，怎么对应到准则

八到十二个。一位 2.75 年经验、2025 年接了 offer 的 Amazon SDE-II 候选人在 LeetCode Discuss 上写的复盘，给出的也是同一个量级：十个到十五个故事，每个打上两三条准则的标签。[^6] 比这更少，同一个项目就会在几轮里反复出现，Bar Raiser 在 debrief 里一对就给你扣分。比这更多，你根本练不过来。

八到十二这个数是能用的，因为够强的故事本来就干净地覆盖两三条准则。一个迁移项目，你先一个人扛了六周，公司才招人进来帮忙——这就是 Ownership（主人翁意识）、Bias for Action（行动优先）加 Deliver Results。线上发布、代码评审都过了之后被你抓出来的那个 bug，是 Dive Deep、Earn Trust（赢得信任）加 Insist on the Highest Standards（坚持最高标准）。每个故事只打它真能证明的准则，别打你希望它证明的。硬凑的对应一听就是背过的，Bar Raiser 受的训练就是抓这个。[^1]

你的故事库要覆盖跟算法轮最贴近、出现最频繁的五条（Customer Obsession、Earn Trust、Dive Deep、Insist on the Highest Standards、Bias for Action），再加上几乎每场行为面都会出现的四条（Ownership、Deliver Results、Are Right A Lot（判断得对）、Have Backbone（敢于坚持））。这一遍过完，要是有哪条准则一个故事都没有，补一个。公司层面那几条准则里有两三条（Strive to be Earth's Best Employer（努力成为地球上最好的雇主）、Success and Scale Bring Broad Responsibility（成功与规模带来广泛责任））在 SDE 面试里几乎不出现，各备一个故事当保险就够了。

## 哪四个错误会毁掉一个纸面很强的人

**通篇“我们”、没有一句“我”的故事。** 这是最常见的翻车方式。候选人讲了一个真实、也确实有分量的项目，但全程用团队的口气在说。面试官的 debrief 上写的是“无法确认候选人的具体贡献”，这一轮分数就下来了。改法：凡是你本人做的动作，把“我们”改掉。只有动作真是集体做的，才留“我们”。[^2]

**有结果，没指标。** *“效果挺好的，团队也很满意”* 不算结果。Bar Raiser Dawn Brun 公开发过的建议在这点上很直接：每个故事都要有一个可量化的结果。[^7] 前面提到的那篇 2026 年 loop 拆解把“故事没有可量化结果”列为行为面掉分的前三大原因。[^1] 改法：每个故事都用数字收尾。延迟、金额、错误率、省下的时间、留存、人数，什么能量化就报一个。

**故事太老。** 一个资深候选人讲大学实习的故事，会触发另一种警报：这八年里再没发生过比它更强的事。故事尽量从最近两三年里选；更早的留一个也行，前提是它确实是你最强的那一个，但它不该是整库的重心。[^1]

**不跟着问题走。** Bar Raiser 问 *“讲一次你跟主管意见不一致的经历。”* 候选人张口就开始讲自己准备最熟的那个 Customer Obsession 故事，因为那是他最好的一个。面试官记下的是“候选人没有回答问题”。Bar Raiser Josh Hirschland 和 Jess Turner 都把这个模式标成扣分项。[^7] 改法：故事库同时按准则和按问题形状打标签，选那个能回答被问到的问题的故事，哪怕它不是最强的一个。

## 本周做什么

写下最近三年的十个项目，每个一段。每个都列出：它真能证明哪几条准则、Result 里的那个指标是什么、你负责的*具体*动作是什么。你会发现有两三个初稿过不了这一关（没有指标，或者归属不清楚）。换掉它们。

然后掐着表出声练。每个故事四分钟，不看稿。盯着 Situation 的时长：超过一分钟就是太长。第一次录，你会讨厌那段录音。第二次，勉强能听。第三次，它就是你周五要讲的那一版。

至于 Amazon 编码轮那一层的讲述（45 分钟解题里穿插的五个时刻，不是完整故事），[编码面试中的沟通](./02-communicating.md) 是更完整的基线。Meta 那边对应的做法格式差别很大，[Meta 的 AI 辅助编码轮](./08-meta-ai-round-format.md) 是那篇对照文。两家公司打分的东西很不一样，准备时别混。

## 参考文献

[^1]: Rafay Abbasi, "Inside the Amazon 2026 Loop: Rounds, Rubric, and What Each Interviewer Scores You On," *Cracking the Tech Interview* (Substack), May 7, 2026, [dglearning.substack.com/p/inside-the-amazon-2026-loop-rounds](https://dglearning.substack.com/p/inside-the-amazon-2026-loop-rounds).

[^2]: Brittany Bunch, "Your complete guide to the Amazon interview process," *About Amazon* (Workplace), April 8, 2025, [aboutamazon.com/news/workplace/amazon-interview-guide](https://www.aboutamazon.com/news/workplace/amazon-interview-guide).

[^3]: Janz, T., Hellervik, L., and Gilmore, D.C., *Behavior Description Interviewing: New, Accurate, Cost Effective*, Boston: Allyn & Bacon, 1986, ISBN 0-205-08597-0.

[^4]: Flanagan, J.C., "The Critical Incident Technique," *Psychological Bulletin*, 51(4), 327-358, 1954.

[^5]: Amazon Jobs, "Leadership Principles" (canonical page), [amazon.jobs/content/en/our-workplace/leadership-principles](https://www.amazon.jobs/content/en/our-workplace/leadership-principles)

[^6]: "Amazon SDE II Final Loop Experience by a 2.75 YOE candidate," *LeetCode Discuss*, 2025, [leetcode.com/discuss/post/6706138/amazon-sde-ii-final-loop-experience-by-a-svnh/](https://leetcode.com/discuss/post/6706138/amazon-sde-ii-final-loop-experience-by-a-svnh/).

[^7]: Alexis L. Loinaz, "8 interview tips from Amazon Bar Raisers to help you clinch the job," *About Amazon* (Workplace), [aboutamazon.com/news/workplace/amazon-job-interview-questions-tips-bar-raisers](https://www.aboutamazon.com/news/workplace/amazon-job-interview-questions-tips-bar-raisers).
