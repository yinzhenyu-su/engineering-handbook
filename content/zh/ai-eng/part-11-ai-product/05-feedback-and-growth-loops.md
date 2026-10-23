---
title: "反馈与增长回路"
description: "大多数团队只有一份日志文件，而不是一个飞轮。一条能跑起来的数据回路需要三个前提，而真正持久的优势来自回路本身，而非数据。"
slug: 05-feedback-and-growth-loops
part: 11
chapter: 5
difficulty: advanced
prerequisites:
  - part-4-eval-fundamentals/01-look-at-your-data
  - part-8-evaluation-and-observability/08-feedback-and-the-data-flywheel
  - 02-ai-product-metrics
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# 反馈与增长回路

Bloomberg 用自家规模可观的专有语料训练了一个面向金融领域的 LLM，取名 Bloomberg GPT，发布时在金融基准上打赢了 GPT-3。一年之内，GPT-3.5 和 GPT-4 就把它盖过去了。[^1] 这件事的教训不是微调没用，而是一次性的数据优势会随着基础模型进步而衰减。真正持久的优势是另一种形态：把产品设计成用户行为持续产生信号、近乎实时地改进 AI，形成一种随规模扩大而拉开的优势，也就是增长回路（growth loop）。[^1]

这就是飞轮。大多数 AI 团队没有飞轮，他们只有一份日志文件。

这个区别之所以要紧，是因为“在攒数据”和“在复利式改进”之间的差距是结构性的，不是技术性的。Tian Pan（2026 年 5 月）列出了三个前提，必须同时成立，回路才真的会复利：反馈信号必须有效，回路必须从用户行为一路闭合到线上变更，反馈延迟必须短到能跑赢基础模型的迭代周期。[^2] 任何一个断掉，复利就停了。

本章讲的是这条回路在产品战略层面的那一面。错误分析、标注流水线、裁判对齐这些技术机制，在[反馈与数据飞轮](../part-8-evaluation-and-observability/08-feedback-and-the-data-flywheel.md)那一章。这里要回答的是：为什么拥有这条回路本身就是竞争优势，真正的飞轮和一份问卷差在哪里，以及该埋哪些点才能让它真的转起来。

## 三个前提

**信号有效性。** 反馈信号必须与目标指标有因果关系。用户点了一下代码建议，不等于这段代码是对的。一个“踩”可能表示“答案错了”，也可能表示“内容冒犯”，或者“我今天不想看这个话题”，这类互相矛盾的信号即使越攒越多，平均下来也会让行为变差。检验方法很直接：如果模型从这个信号里学习，它在目标指标上的行为真的会变好吗？如果你拿不出证据回答“是”，这个信号就是戴着面具的噪声。

**回路闭合。** 数据必须在物理上从用户行为流经采集、标注、改进，最后落到一次线上变更。这话听着理所当然，直到你去审计组织，才发现运维把 LLM 交互日志打到 Datadog，ML 团队从另一个数据仓库做微调，没人把两头接起来。Gartner 2025 年对 248 位数据管理负责人的调查发现，63% 的组织要么缺少、要么不确定自己有没有适配 AI 的数据管理实践。[^3] 这个缺口是结构性的：运维管日志，ML 管训练数据，生产信号就在两者的缝隙里蒸发了。[^4]

**反馈延迟足够短。** 如果从用户行为到线上改进的周期超过 30 天、中间又没有阶段性验证，团队就不可能比基础模型迭代得更快，也跑不赢做得到的竞争对手。[^2] Tesla 一个晚上就处理完车队的接管事件（disengagement）。多数应用型 AI 团队用的还是一个月前的日志。按周迭代的团队一年复利 52 次，按月迭代只有 12 次。差距是几何级拉开的。

![一个四阶段的环形回路图：用户行为流向信号采集，再流向模型改进，再流向更好的输出，最后回到用户行为，循环明确闭合，并标注为复利](../../../ai-eng/part-11-ai-product/images/05-growth-loop.png)
*护城河是回路，不是数据。每转一圈，下一圈都更容易。*

## 隐式信号才是量的来源

显式反馈（点赞、星级、评分）在生产环境的 LLM 部署里平均只占全部交互的 0.5% 到 5%。[^5] 1,000 日活、1% 的显式反馈率，一天也就收 10 到 30 条带评分的样本。要有统计把握地检出 5% 的质量变化，大约需要 1,000 个样本，也就是说得等 30 到 100 天，信号才攒得起来。

隐式行为信号能覆盖全部用户，而且不需要用户在界面上多做任何动作。以代码生成工具为例：内联建议的采纳与拒绝、采纳之后的编辑距离（采纳后又改了 50 个字符以上，算弱负例）、立刻改写提示词重新请求（强失败信号）、采纳的代码几分钟内被回滚（事后失败信号）。聊天和搜索场景则是：追问式改写原问题（第一次回答没答到）、复制粘贴行为（高价值内容才会被复制）、回答后直接离开会话、几小时后又回来问同一个话题。

“重新生成”按钮是音量最大的隐式“踩”：摩擦低、语义不含糊，用户明确选择了不改也不留。[^6] 编码助手和 Midjourney 这类图像生成工具就是围绕这个洞察设计界面的：采纳＝强正例，修改＝正例，忽略或重新生成＝负例，不需要额外点一下。

另一个该倚重隐式信号的理由是：显式信号在低量时会骗人。Morandi 和 Viswanathan 的论文（2026 年 5 月）说明，对点赞／点踩反馈做朴素平均，结果可能偏离真实系统质量 40 到 50 个百分点，因为愿意给反馈的用户在满意度分布的两端上占比过高。一个有记录的生产部署里，朴素算法算出的质量是 34%，还原后的真实质量是 83%，差了 49 个百分点。[^5]

```python
from dataclasses import dataclass

@dataclass
class InteractionBatch:
    total_interactions: int
    rated_interactions: int
    positive_ratings: int

    @property
    def feedback_rate(self) -> float:
        if self.total_interactions == 0:
            return 0.0
        return self.rated_interactions / self.total_interactions

    @property
    def naive_quality(self) -> float:
        if self.rated_interactions == 0:
            return 0.0
        return self.positive_ratings / self.rated_interactions

def selection_bias_warning(batch: InteractionBatch) -> dict:
    if batch.feedback_rate >= 0.05:
        return {"warning": False, "naive_quality": round(batch.naive_quality, 3)}
    return {
        "warning": True,
        "feedback_rate": round(batch.feedback_rate, 4),
        "naive_quality": round(batch.naive_quality, 3),
        "action": "Selection bias risk: corroborate with implicit behavioral signals.",
    }
```

默认架构是这样：隐式信号作为主要的量来源，显式二值评分（通过／不通过，绝不用李克特量表）用来校准那一小部分愿意互动的用户的真值。永远不要只靠显式评分、不做偏差校正就去衡量绝对质量。

## 不用翻倍人力也能闭合回路

行为信号采到之后，瓶颈就转到标注上。生产规模的纯人工标注在经济上不成立。弱监督（weak supervision，Snorkel，Ratner 等，VLDB 2017/2020）是务实的答案：不逐条标注样本，而是写一批标注函数，按模式给样本分类，再用一步去噪把多个有噪声的函数合起来。[^7]

给编码助手写的标注函数可以是这样：用户采纳了建议、10 分钟内没有回滚，标为正例。另一条：用户把建议的代码改掉了 50% 以上，标为负例。Snorkel 的实验结果：领域专家建模快 2.8 倍，相比 7 小时人工标注平均把预测效果提升 45.5%，与大规模人工精标训练集的效果差距在 3.60% 以内。[^7] 反直觉的结论是：10 万个不完美的标签，通常好过 100 个完美标签。量和覆盖度比单条标签的精度更重要，因为数据够多时模型会把噪声平均掉。

标注好的数据流向两条改进路径。微调带来持久收益，但需要重训基础设施、评测门禁和部署流水线，周期以周计。动态 few-shot 检索更快：维护一个带指标分数和人工修复结果的生产 trace 库；推理时检索最相似的已修复 trace，作为 few-shot 示例塞进提示词。[^6] 以小时到天计，不是以周计。提示词工程于是变成一个检索问题。天花板更低（这种方式改不了模型本身的根本局限），但节奏快到多数团队应该默认走这条路，只有在撞到 few-shot 天花板时才升级到微调。

这条回路最强的变体是带可验证奖励的强化学习：系统生成输出，自动验证正确性（代码能编译、测试能过、SQL 能执行），不等人工反馈就能改进。它只适用于真值可以被机器检查的领域，但在适用之处，回路转得和你跑推理一样快。Duolingo 的 Birdbrain 在学员每做对或做错一道题时就更新对学员能力和题目难度的估计，一天大约 10 亿道题；第二版把这些更新从每晚批处理缩短到几分钟内完成。[^8] 信号（每道题对或错）是可机器验证的，不需要标注。

## 规模化之后长什么样

GitHub Copilot 的 signals loop（2025 年 10 月）是规模化端到端回路闭合的公开参考。最新的代码补全模型用公开仓库里 40 多万条真实样本训练，再用合成训练数据做 RL 微调。配合客户端和 UX 的改动，最终代码补全的留存代码量提升 30% 以上、速度提升 35%，而这个产品的用户已超过 2,000 万。[^9] 微软对这次架构转向的表述是：“随着基础模型日益商品化，AI 产品的长期防御力不会只来自模型本身，而来自这些模型从使用中学习的效率。”[^9]

微软用于临床听写的 Dragon Copilot 走的是相似的路，但评测门禁更严，因为出错会牵涉患者安全。每个新的基础模型都要用自动化指标跑一遍基准，只有提升显著才采纳。团队在临床数据仓库上做微调，之后持续用客户反馈遥测精修模型；微软称最新模型比基础模型好约 50%。[^9] 医疗正是那种“结构化的显式反馈值得付出这份摩擦”的场景。

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class LoopEvent:
    loop_id: str
    signal_captured_at: datetime
    deployed_at: Optional[datetime] = None

    def total_loop_hours(self) -> Optional[float]:
        if self.deployed_at is None:
            return None
        return (self.deployed_at - self.signal_captured_at).total_seconds() / 3600

def loop_health(events: list[LoopEvent]) -> dict:
    closed = [e.total_loop_hours() for e in events if e.total_loop_hours() is not None]
    if not closed:
        return {"n": 0, "median_loop_hours": None}
    closed.sort()
    return {
        "n": len(closed),
        "median_loop_hours": round(closed[len(closed) // 2], 1),
        "loops_under_168h": sum(1 for h in closed if h < 168),
    }
```

唯一值得埋点的健康指标：信号到部署延迟的小时中位数。快速迭代的 AI 产品目标是在 168 小时（7 天）以内。超过 720 小时（30 天）的回路，在基础模型的改进把这一轮迭代作废之前，根本检不出 5% 的质量变化。

## 让回路停转的几种失败模式

**点赞陷阱。** 加一个反馈组件，盯着点赞率，只要还停在 80% 以上就宣布模型在变好。前面 Morandi 和 Viswanathan 的选择偏差例子：朴素值 34%，真实值 83%。[^5] 显式评分永远要和隐式基线（重试率、复制率、放弃率）交叉验证，绝不要把朴素的点赞比例当成绝对质量指标。

**退化回路。** 模型在自己的输出上训练，一轮轮放大已有偏差。推荐系统把这个模式演得很清楚：用户点什么取决于排序，模型于是学会复现自己的排序。Pinterest 在推荐漏斗上发现了“富者愈富”效应，老内容压过新内容，为此专门建了去偏系统来对冲。[^11] 在生成式 AI 里，用户采纳了一封 AI 起草的邮件，并不等于这封邮件写得好，他可能只是赶时间。模型学到的是“写出让心不在焉的用户愿意采纳的邮件”。防御手段是探索（定期拿出低置信度的预测、收集显式反馈）和延迟的结果信号（邮件发出去了吗？对方回了吗？），而不是“当场采纳”这种即时信号。

**组织断层。** 生产信号采了，却从没改进过模型，因为从用户事件到线上变更这条路没人负责。这是结构问题，不是技术问题。解法是明确归属：由一个人或一个团队负责端到端这条路径，事件 schema、标注流水线、评测门禁、部署流水线共用同一套 schema 和 join key（每条标签都带 request ID），这样 trace 之间才能相互对上。[^4]

**合成数据导致的模型崩塌。** Shumailov 等人的论文（Nature，2024 年 7 月）证明，“在训练中使用模型生成的内容，会给模型带来不可逆的缺陷，原始内容分布的尾部会消失”。[^10] LLM 生成的标签在反馈回路里尤其危险，因为它出自正在被改进的那个模型，回路变成自指的。每次微调都要保住一定比例的人工标注数据；把 LLM 标签当作弱监督、与人工标签合并使用，而不是当成独立的真值。

**头三个月的平台期。** 团队把基础设施搭好、上线、盯着回路看了三个月，没看到提升，就把项目废了。这其实是校准期，不是失败。1,000 日活、1% 显式反馈率下，要以统计方式检出 5% 的变化，需要 30 到 100 天的积累。上线前先算出“到达显著性所需时间”；显式量不够，就从第一天起用隐式信号和弱监督补上。前 30 天用来验证标注流水线，不是用来检测质量变化。

## 复利优势及其边界

拥有回路的团队会攒下三样东西，竞争对手不付同样的使用成本就复制不了。领域校准：只在生产里才冒出来的失败模式（奇怪的措辞、边角输入、领域术语）会沉淀进你的评测集。评测集成熟度：评测标准是靠看数据发现的，不是事先规定好的，一个成熟的评测集代表几个月真实生产故障的积累，这些东西在干净的实验室里复现不出来。迭代速率：按周迭代一年复利 52 次，按月只有 12 次，差距是几何级拉开的。

诚实的提醒是这个护城河有边界。Bloomberg GPT 有真正的专有数据，发布表现也强，基础模型的进步还是在一年内抹平了它的优势。飞轮能在同一代模型内制造复利，但它不给你对基础模型颠覆的免疫。[^1] 防御办法是让回路穿过模型升级持续运转：新基础模型落地时，用你持续更新的生产数据集重训，而不是把微调当成一次性事件。

竞争上的结论依然成立。静态数据不是护城河，动态回路才是。按周迭代的团队积累的优势，和一季度才微调一次的团队不是同一种东西，而且这种优势体现在迭代速率上，不体现在数据集大小上。

## 参考文献

[^1]: Kartik Hosanagar, "The Real AI Moat: Data Flywheels", SXSW 2026 keynote / Wharton faculty essay, March 2026.

[^2]: Tian Pan, "The Data Flywheel Assumption: When AI Features Compound and When They Just Accumulate Noise", May 5, 2026. https://tianpan.co/blog/2026/05/05/data-flywheel-assumption-ai-features-noise

[^3]: Gartner, "Survey: 63% of Data Management Leaders Lack or Are Unsure of AI-Ready Data Practices", February 2026 press release citing Q3 2024 survey of 248 leaders.

[^4]: Tian Pan, "Closing the Feedback Loop: How Production AI Systems Actually Improve", April 15, 2026. https://tianpan.co/blog/2026/04/15/closing-the-feedback-loop-production-ai

[^5]: Morandi and Viswanathan, "Selection Bias in Production LLM Feedback", arXiv:2605.12177, May 2026. https://arxiv.org/abs/2605.12177

[^6]: Shankar et al., "Applied LLMs: Practitioner's Guide to LLM Production", applied-llms.org, June 2024. https://applied-llms.org

[^7]: Ratner et al., "Snorkel: Rapid Training Data Creation with Weak Supervision", VLDB 2017/2020, arXiv:1711.10160. https://arxiv.org/abs/1711.10160

[^8]: Klinton Bicknell, Claire Brust, Burr Settles, "How Duolingo's AI Learns What You Need to Learn", IEEE Spectrum, February 2023. https://spectrum.ieee.org/duolingo

[^9]: Asha Sharma and Rolf Harms, Microsoft Azure, "The Signals Loop: Fine-tuning for world-class AI apps and agents", October 21, 2025. https://azure.microsoft.com/en-us/blog/the-signals-loop-fine-tuning-for-world-class-ai-apps-and-agents/

[^10]: Shumailov et al., "The Curse of Recursion: Training on Generated Data Makes Models Forget", Nature, July 2024 (originally arXiv:2305.17493). https://www.nature.com/articles/s41586-024-07566-y

[^11]: Gudmundsson et al. (Pinterest), "PinEqualizer: Full Funnel Content Exploration and Debiasing System at Pinterest", KDD 2026, arXiv:2607.22518, https://arxiv.org/abs/2607.22518
