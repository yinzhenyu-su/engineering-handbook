---
title: "按公司划分的线路：索引与用法"
description: "什么时候该打开一份按公司划分的线路，怎么把它叠在第 0 至 13 部分之上，以及三个信号告诉你该从模式练习转向面向具体公司的校准。"
slug: 11-per-company-tracks-index
part: 14
chapter: 11
difficulty: intermediate
prerequisites:
  - part-14-interview-framework/00-pattern-recognition
  - part-14-interview-framework/04-mock-interview-process
  - part-14-interview-framework/05-common-pitfalls
date_updated: 2026-05-25
---

# 按公司划分的线路：索引与用法

按公司划分的线路叠在核心手册之上，不是它的替代品。如果没有第 0 至 13 部分和[模式识别](./00-pattern-recognition.md)练出来的模式识别能力，按公司线路就只是一串贴了公司标签的 LeetCode 题，而这恰恰是 2024 至 2026 年的证据显示会带来脆弱结果的备考底料。

三个信号告诉你该开一份线路了。招聘人员的电话已经约进日历。你的面试流程（loop）落在十二周之内。或者你手上有多个 offer 需要校准。没有其中之一的话，花在线路上的每一个小时都是从你尚未见过的模式里挪走的，这一个小时花在核心书上更值。

信号一旦触发，问题就变成选哪条线路。下面六家公司各配一句话定位、线路所围绕的瓶颈维度，以及真正动手训练所在的更细子章节。

## Amazon

行为面权重是 FAANG 阶梯里最高的。现场面试的编码轮总有一到两个 Leadership Principle 问题，而 Bar Raiser 的说法是：行为面强可以救编码弱，编码强救不了行为面弱。[^1] 这条线路把 LP 陈述前置到任何 LeetCode 练习之前。2025 和 2026 的面试流程里禁止使用 AI 工具。[^1]

深入看[Amazon Leadership Principles 简报](./06-amazon-lps-brief.md)和[在 DSA 轮里讲 LP](./07-amazon-lps-narration.md)。

## Meta

远程面试以来，大厂 DSA 面试最大的单项形式变化：有一轮编码（到 2026 年正在向所有轮次铺开）是 AI 辅助的，CoderPad 内置助手提供 GPT-5、Claude Sonnet 4.5、Gemini 2.5 Pro 和 Llama 4 Maverick。[^2] 节奏是另一个瓶颈。电话筛选是 35 分钟两道题。按 Meta 内部面试官指引，DP 被禁用。

[Meta 的 AI 辅助轮](./08-meta-ai-round-format.md)覆盖提示词模式、模型菜单，以及 35 分钟两道题的电话筛选节奏。

## Google

编码的权重高于系统设计，是 FAANG 里唯一如此的一家。[^3] DP 完全在考查范围内，也是 FAANG 里最可能考 DP 的一家（Yangshun：“如果进 Google 是你的梦想，DP 就躲不掉”）。要预期误导性表述，以及会推翻前面假设的追问。Googleyness 轮依据的是普遍被报告的素质而非官方公布的条目，用来评估文化契合度。[^4]

按公司线路（计划中）：`content/dsa/per-company/google.md`。

## Apple

FAANG 阶梯里团队间差异最大的一家。[^5] 一个团队考经典的 LC 中等题；另一个团队是 take-home 加现场调试；还有一个团队会花一整天深挖 Java 并发或 Swift 内部机制。“Why Apple?” 在行为面里是生死题，多个面试官仅因这一题答得弱就否决候选人。心里没有具体团队之前，不要打开 Apple 线路。

按公司线路（计划中）：`content/dsa/per-company/apple.md`。

## Microsoft

六家里最“经典 CS 基础”的流程。高频标签题偏向简单到中等，常见解析、IP 校验、整数转英文单词、链表操作。[^6] DP 少见但在渗入（换零钱、编辑距离这个级别）。有一轮是领域专项，考对该团队技术栈的深度：云网络、大数据、分布式存储。“As Appropriate” 面试官做最终的录用/不录用决定。

按公司线路（计划中）：`content/dsa/per-company/microsoft.md`。

## Netflix

Netflix 之于系统设计，就像 Google 之于编码。[^7] 编码在 FAANG 里权重最低；设计和行为面权重最高。流程大约八场面试，常常分两天进行，面试小组里有一到两位总监。总监主导的 “Dream Team” 行为轮可以单独把候选人否掉。Netflix 只招资深，门槛是设计加行为面，招聘量很小。

按公司线路（计划中）：`content/dsa/per-company/netflix.md`。

> [!WARNING]
> **对比里的两个格子漂移得很快。** Meta 的 AI 轮形式是全书最新的信号（2025 年 10 月落地，2026 年持续扩大）；面试流程之前先对着 [interviewing.io 的博客](https://interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples)核一下日期。很多读者会去翻的 krishnadey30 LeetCode-CompanyWise，最后一次有意义的更新是 2022 至 2023 年；把它当成相对权重看，绝不要当成绝对频率。[^8]

## 本周做什么

如果你中了三个信号之一，打开对应的线路，把它当作接下来四到十二周备考的主干。否则，关掉这一页，回去把核心书里你为了走到这里而跳过的那部分补完。线路的价值体现在校准阶段，它们替代不了核心。

[常见陷阱](./05-common-pitfalls.md)是横切各处的失误清单。[模拟面试流程](./04-mock-interview-process.md)讲的是怎么模拟上面六个瓶颈维度中的任意一个。接下来 6 到 12 周的备考，才是决定这场面试流程的部分。

## 参考文献

[^1]: interviewing.io, "Amazon interview process and questions," refreshed Q4 2025, [interviewing.io/guides/hiring-process/amazon](https://interviewing.io/guides/hiring-process/amazon).

[^2]: interviewing.io blog, "How to Use AI in Meta's AI-Assisted Coding Interview, with Real Prompts and Examples," published 2025, [interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples](https://interviewing.io/blog/how-to-use-ai-in-meta-s-ai-assisted-coding-interview-with-real-prompts-and-examples).

[^3]: interviewing.io, "Google interview process and questions," refreshed 2026, [interviewing.io/guides/hiring-process/google](https://interviewing.io/guides/hiring-process/google).

[^4]: Hello Interview, "Google L5 (Senior) Software Engineer Interview Guide," 2025, [hellointerview.com/guides/google/l5](https://www.hellointerview.com/guides/google/l5).

[^5]: interviewing.io, "Apple interview process and questions," [interviewing.io/guides/hiring-process/apple](https://interviewing.io/guides/hiring-process/apple).

[^6]: interviewing.io, "Microsoft interview process and questions," [interviewing.io/guides/hiring-process/microsoft](https://interviewing.io/guides/hiring-process/microsoft).

[^7]: interviewing.io, "A Senior Engineer's Guide to Netflix's Interview Process and Questions," by Kevin Landucci, fetched 20 May 2026, [interviewing.io/netflix-interview-questions](https://interviewing.io/netflix-interview-questions).

[^8]: krishnadey30, "LeetCode-Questions-CompanyWise" GitHub repository, [github.com/krishnadey30/LeetCode-Questions-CompanyWise](https://github.com/krishnadey30/LeetCode-Questions-CompanyWise).
