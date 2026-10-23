---
title: "AI 工程师是什么（以及不是什么）"
description: "AI 工程师、ML 工程师与研究员的区别：API 分界线、三种角色，以及这本手册到底教什么、跳过什么。"
slug: 00-what-an-ai-engineer-is
part: 0
chapter: 0
difficulty: beginner
prerequisites: []
tags: [llm]
date_created: 2026-06-11
date_updated: 2026-06-11
---

# AI 工程师是什么（以及不是什么）

2023 年 6 月 30 日，Shawn Wang（笔名 swyx）发表了一篇题为“The Rise of the AI Engineer”的文章，给一个早已存在却没有名字的岗位起了名。他的论证很简单：全世界大概有 5,000 人在训练前沿模型，而能写软件的有 5,000 万人左右。模型刚刚好到这样一个程度——第二群人可以在第一群人的成果之上做出真正的产品，完全不用重新训练任何东西。总得有人去填这道缝。

OpenAI 创始成员 Andrej Karpathy 用一句话划出了边界：“One can be quite successful in this role without ever training anything.”[^1]

你不必训练模型，也能擅长用模型做东西。

## API 分界线

整个技术栈分成两块地盘，分界线就是模型 API。

**线以下是训练模型的团队。** 他们收集预训练数据、跑 GPU 集群、调 loss 曲线、发布权重。那是 ML 工程师、研究工程师和研究科学家的世界。

**线以上是用模型的团队。** 他们调用 API（或者自托管一个开放权重模型）、拼装模型看到的上下文、把它接到工具上、评估输出，再把结果塞进产品里交付。你会在这一侧工作。

swyx 把这条边界叫作 API 分界线（API line），并且特意强调它是可渗透的：AI 工程师有时也会微调一个小模型，研究工程师有时也会在 API 之上做东西。[^1]

Chip Huyen 的书 *AI Engineering*（O'Reilly，2024 年 12 月）是这个领域最完整的论述，她用另一种方式划了同一条线。她把右侧那一半叫作“模型即服务（model-as-a-service）”路线，说这一转变“把 AI 从一门晦涩的学科变成了任何人都能使用的强大开发工具”。[^2] 她的书用整整一章讨论这条边界：“AI Engineering Versus ML Engineering”。

这条线会渗漏，但它仍然是重心所在。AI 工程师可以一个权重都不改，就做到很出色。他们真正负责的是：模型行为、上下文窗口、评估，以及生产运行。

![一条虚线靛蓝色边界把两个区域分开，边界上标着 API 分界线：下方是一位研究员和一位 ML 工程师，旁边立着服务器机柜；上方是一位 AI 工程师，抱着笔记本，周围漂浮着工具卡片](../../../ai-eng/part-0-start-here/images/00-the-api-line.png)
*线以下的一切都在改动模型权重；线以上的一切都通过 API 消费它们。*

有多少工作真的落在线上那一侧？一份针对 2026 年 1 月 1,000 多份 AI 工程师职位描述的分析发现，这个角色压倒性地偏向生产。只有约 4% 把微调列为主要职责，而大约 81% 的招聘启事压根没提微调。[^3] 这只是一份有日期的研究，不是什么自然定律，但它指的方向和当初给这个角色命名的人一致。

## 三个角色，各一句话

| 角色 | 负责什么 | 迭代速度 |
|---|---|---|
| **AI 工程师** | 基础模型之上的产品：提示词、上下文、检索、智能体（agent）、评估、服务、成本。 | 提示词按小时算，检索按天算。 |
| **ML 工程师** | 自己或团队训练的模型与模型流水线：训练循环、特征库、重新训练。 | 每次重训几周到几个月。 |
| **研究员** | 新架构、新训练方法、新能力，以论文和权重的形式发布。 | 每项进展几个月到几年。 |

把这张表从左往右读，你会看到一条供应链。研究员发明模型。ML 工程师把团队训练出来的模型投入运行。AI 工程师则在通过 API 出来的任何东西之上做产品。

边界在真正管用的地方是实的：面试流程、职级、薪酬区间都不一样。而最干净的判断标准只有一个问题：你的主要工作会改动模型权重吗？会，你就是 ML 工程师。不会，你就是 AI 工程师。

## 这个角色真的存在吗？

持怀疑态度的人到这里会反驳，而且反驳得有理。有人说“AI 工程师”只是头衔通胀，是全栈或 ML 岗位上贴的新标签。一份关于企业招聘的批评指出，大约 75% 的候选人在结构化的 AI 工程评估中不合格，而公司把三类不同的工作塞进同一份职位描述里。[^4] 这种抱怨并不荒唐。调用 API 的门槛低，就会吸引来只会调用 API 的人。

市场仍然给出了回答。LinkedIn 在 2026 年 1 月发布的 Jobs on the Rise 2026 榜单中，把“AI 工程师”列为美国增长最快的职位第一名。[^5] 根据 levels.fyi，截至 2025 年第三季度，这个岗位在美国的总薪酬中位数约为 245,000 美元——一个有日期的数据点，不是保证。[^6] 公司现在为它专门设计了面试流程。头衔很乱，岗位是真的。

## 当你的依赖是一个模型，什么变了

你的大部分工程直觉都能迁移过来。[软件基线](./01-the-software-baseline.md) 列出了本书假定你已经具备的技能。

有一个直觉迁移不了。你现在最重要的依赖是概率性的：同样的输入可能返回不同的输出，你的测试套件没法断言精确字符串。[概率系统思维](./02-thinking-in-probabilistic-systems.md) 整篇内容都从这一个事实展开。

你构建的方式也有对应的转变。本书讲一架从朴素提示词一路通到微调的阶梯，每一级都比上一级更贵。大多数功能永远爬不过第三级。[升级阶梯](./03-the-escalation-ladder.md) 是你每周都会用到的决策工具。

## 这本手册教什么（以及跳过什么）

**它教**怎么用模型 API、怎么写提示词、评估先行的开发方式、上下文工程、检索、智能体、可观测性、安全、成本工程，以及如何交付用户能信任的 AI 功能。一切都对着生产：真实的预算、真实的故障模式、真实的用户。微调作为最后手段在范围内；Part 10 讲什么时候该用它，以及更常见的情况——什么时候不该用。

**它跳过**从零训练模型、预训练基础设施和架构研究。分词、采样这类模型内部细节，只在它们作为你真正会去拧的成本、延迟或质量旋钮出现时才讲。Huyen 划的是同一条线。swyx 说得更直白：“When it comes to shipping AI products, you want engineers, not researchers.”[^1]

这条分界也是对“什么能持久”的一次下注。预训练知识随着前沿推进几个月就过期；评估和运营基于模型的系统这套技能会不断复利。

如果你想*造*模型，这本书是错的选择，Huyen 更早的 *Designing Machine Learning Systems* 才合适。如果你想专业地**用**模型做东西，继续读下去。

## 参考文献

[^1]: swyx (Shawn Wang), "The Rise of the AI Engineer," Latent Space, June 30, 2023. https://www.latent.space/p/ai-engineer
[^2]: Chip Huyen, "AI Engineering: Building Applications with Foundation Models," O'Reilly Media, December 2024. ISBN 9781098166298. https://www.oreilly.com/library/view/ai-engineering/9781098166298/
[^3]: Alexey Grigorev and Valeriia Kuka, "What 1,000+ Job Descriptions Reveal About the AI Engineer Role in 2026," Alexey on Data, March 4, 2026. https://alexeyondata.substack.com/p/what-1000-job-descriptions-reveal
[^4]: Kennith Jackson, "Why hiring 'AI engineers' won't work," CIO.com, April 23, 2026. https://www.cio.com/article/4162080/why-hiring-ai-engineers-wont-work.html
[^5]: LinkedIn News, "LinkedIn Jobs on the Rise 2026: The 25 fastest-growing roles in the U.S.," LinkedIn Pulse, January 7, 2026. https://www.linkedin.com/pulse/linkedin-jobs-rise-2026-25-fastest-growing-roles-us-linkedin-news-dlb1c
[^6]: Alina Kolesnikova, "AI Engineer Compensation Trends Q3 2025," levels.fyi, July 17, 2025. https://www.levels.fyi/blog/ai-engineer-compensation-trends-q3-2025.html
