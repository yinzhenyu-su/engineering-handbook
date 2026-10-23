---
title: "实验"
description: "当方差比确定性功能高 2-5 倍、处理组会在实验中途漂移，而读出来没有显著效果通常意味着功效不足时，怎么给 AI 功能做 A/B 测试。"
slug: 03-experimentation
part: 11
chapter: 3
difficulty: advanced
prerequisites:
  - part-4-eval-fundamentals/02-your-first-eval-set
  - part-8-evaluation-and-observability/05-online-evaluation-and-ab-testing
  - 02-ai-product-metrics
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# 实验

模型团队能拿出十个定性层面的胜例，一条一条摆给你看。增长团队把同一个改动做成两周的 A/B 测试，跑出来 p=0.31，结论写的是“无显著效果”。两个团队都没错。[^1] 错的是这个实验。

这种错位来自一个事实：经典频率学派（frequentist）的 A/B 数学建立在两条假设上，它们对按钮颜色测试成立，对 LLM 功能则不成立：单个用户的方差相对于处理效应很小，每次测量大致是用户行为分布的一次独立抽样。LLM 功能两条都违反，实际后果是，把确定性功能的方差喂给标准样本量计算器，返回的数字会差 2-5 倍。要拿到同样的统计功效（statistical power），团队需要多出这么多倍的样本量或实验时长。[^1]

本章讲怎么给 AI 功能设计不会骗你的 A/B 测试，怎么诚实地读一个不确定的结果，以及什么时候离线评估就够了、什么时候必须上线上实验。

## 为什么 LLM 的方差天生更高

2026 年 1 月的一项研究（12 个 LLM、10 条提示、N=12,000 条输出）给出了最干净的实证拆解：就输出质量而言，提示解释了 36.43% 的方差，模型选择解释了 40.94%。就输出数量而言，模型内部的采样方差本身就解释了 33.70%。作者明确警告：“单样本评估有把采样噪声与真实的提示或模型效应混为一谈的风险。”[^2] 即使在 temperature 0 下，非确定性依然存在，原因包括服务商的批处理、模型快照，以及硬件层面的矩阵乘法顺序。[^3]

这个含义很具体。在一个 64 例的评估集上，2023 年的 GPT-4 准确率在连续多次完全相同的运行之间会在 65% 到 75% 之间波动。[^4] 在这个集合上观察到从 73% 涨到 76%，完全落在单个随机模型的噪声带里。在同一 64 例集合上比较两个提示变体而不做多次重复运行，结论不可靠，没有例外。

样本量计算能把差距摆到明面上。在 82% 的基线上检测 3 个百分点的提升（alpha=0.05，power=0.80），每组大约需要 2,400 个样本。检测 5 个百分点需要约 870 个，10 个百分点需要约 210 个。[^1] 一份典型的人工精编评估集有 50-200 个样本，这意味着 N=100 时最小可检测效应（minimum detectable effect, MDE）约为 11-12 个百分点。团队跑 100 个样本、看到 3 个点的改进，在统计上什么都没证明。

```python
import math
from scipy.stats import norm

def sample_size(p1, p2, alpha=0.05, power=0.80):
    z_a = norm.ppf(1 - alpha / 2)
    z_b = norm.ppf(power)
    var = p1 * (1 - p1) + p2 * (1 - p2)
    return math.ceil((z_a + z_b)**2 * var / (p1 - p2)**2)

def mde(n_per_arm, p_baseline, alpha=0.05, power=0.80):
    z_a = norm.ppf(1 - alpha / 2)
    z_b = norm.ppf(power)
    var = p_baseline * (1 - p_baseline)
    return (z_a + z_b) * math.sqrt(2 * var / n_per_arm)

# 82% 基线上提升 3pp：每组约 2,394 个样本
print(sample_size(0.82, 0.85))
# N=100 时，你只能检测到 11.4pp 的提升
print(f"{mde(100, 0.82):.1%}")
```

## 设计实验

改提示、换模型、调 temperature、改检索配置，都是合法的 A/B 处理，但一次只能动一个。如果同时改提示和模型，结果就归因不到任何一方身上。[^5] Statsig 的框架区分三类处理，并建议各自单独立实验：提示变体、模型选择、参数调整。[^5]

随机化单元几乎永远应该是用户，而不是会话或请求。用户级随机化避免了携带效应（carryover）：一个用户周一看到处理组、周三看到对照组，期望就乱了。Khan Academy 在会话线程（chat thread）级别做随机化，用跨线程的不一致换更高的统计功效，但 GrowthBook 把它标为例外而非常规起点：随机化单元要与体验单元对齐。[^6] 用户共享账号的 B2B 产品必须在组织级别做聚类随机化（cluster randomization），这会大幅削减有效 N，需要仔细做功效规划。

成功指标要预先注册，并且从正确的层级里选：

| 层级 | 指标类型 | 为什么 |
|---|---|---|
| 1 | 下游业务结果（转化、续费、完成率） | 最可信、最难刷、灵敏度最低 |
| 2 | 行为代理指标（采纳率、重试率、复制率） | 灵敏度中等，无需显式反馈就能暴露偏好 |
| 3 | 质量评分（点赞、星级） | 样本自选、正性偏差、失败模式被混在一起 |
| 护栏 | 安全底线（幻觉率、毒性、PII 泄漏） | 是要守住的约束，不是要优化的指标 |

Intercom 的案例是每个团队都该听的警告。一个离线评估分数*更低*的提示变体，在线上 A/B 里跑出了*更好*的业务结果。Intercom 首席 ML 科学家 Pedro Tabacof 说：“他花了一些时间调提示把这个指标拉回来，然后我们上了 A/B 测试。业务数字不太好。后来我把原始版本，也就是评估分数更低的那个，拿去做 A/B 测试，业务结果反而更好。”[^6] 离线评估负责过滤回退，线上数据做最终裁决。

成熟的做法是三阶段流水线：离线评估在任何线上曝光之前先滤掉明显的回退；线上实验从 1-5% 流量起步并监控护栏指标；全量上线后继续做生产监控，因为即使没有任何显式改动，模型也会随时间漂移。[^6]

## 方差缩减能挣回成本

CUPED（Controlled-experiment Using Pre-Experiment Data，用实验前数据做对照实验）用实验前的协变量减掉实验后方差里可预测的那部分。方差缩减量是 `Var(Y) * (1 - rho^2)`，其中 rho 是同一批用户实验前与实验后指标的相关性。rho = 0.7 在成熟的参与度指标上很典型，对应 51% 的方差缩减，所需实验时长直接减半。[^7]

```python
import statistics

def cuped_adjust(y_post, x_pre):
    mean_x = statistics.mean(x_pre)
    mean_y = statistics.mean(y_post)
    n = len(y_post)
    cov = sum((y_post[i] - mean_y) * (x_pre[i] - mean_x) for i in range(n)) / (n - 1)
    var_x = statistics.variance(x_pre)
    theta = cov / var_x
    y_adj = [y_post[i] - theta * (x_pre[i] - mean_x) for i in range(n)]
    reduction = 1 - statistics.variance(y_adj) / statistics.variance(y_post)
    return y_adj, reduction
```

CUPED 有两个前提：每个用户至少有 7-14 天的实验前历史，且相关性有意义（rho > 0.3 才值得用）。对新用户群体没用，对改动大到过去行为不再能预测未来行为的场景也没用。Tian Pan 记录了一个案例：样本量和效应大小都没变，CUPED 把 p 值从 0.116（不显著）推到 0.000（明显显著）。[^7]

对排序和检索类功能，交错实验（interleaving）是另一个大杠杆。Airbnb 的交错框架（2021 年建成）在搜索排序实验上比 A/B 敏感 50 倍，交错结果与对应 A/B 结果的方向一致率为 82%。[^8] 把它当作快速的第一道闸门：交错实验挑出明确赢家，再跑 A/B 拿绝对效应大小；交错结果说不清，就先迭代再上完整 A/B。代价是：对依赖集合级优化的排序器，交错实验会失效；而且它测的是相对偏好，不是绝对业务影响。

## 处理漂移、新鲜感、锚定

有三种偏差打 AI A/B 测试比打确定性测试更狠。

**处理漂移（treatment drift）。** LLM 服务商会持续更新模型，且不承诺版本稳定。Anthropic 2025 年 8 月的复盘记录了影响最多 16% Claude Sonnet 4 请求的路由错误，而 API 没有任何变更。[^9] GPT-4 的代码执行直通率在 2023 年 3 月到 6 月这三个月里从 52% 掉到 10%，版本号没变。[^9] 如果你的对照组是“模型 X 加旧提示”，而服务商在你 14 天实验的第 4 天给模型 X 打了补丁，处理组就不再是随机化时的那个东西。把每次推理调用的模型版本作为标准列记下来。如果一次静默更新在两个组之间不对称地落地，这个实验直接作废。

**新鲜感（novelty）。** 任何新的 AI 界面在头几天都会拉高参与度，因为它新；AI 功能的新鲜感比 UI 改动更严重，因为输出每次请求都不同。一项追踪用户八个月的研究发现，部分 AI 功能在初期强劲采纳之后，使用量会随时间显著*下降*。[^10] 两周的实验往往整个落在新鲜感尖峰里。按“首次曝光后第几天”切分处理效应；如果提升集中在第 1-3 天，到第 14 天走平甚至转负，那是新鲜感，不是信号。

**锚定（anchoring）。** 用户第一次接触 AI 功能时会形成一个基线期望。处理组用户如果碰巧第一次就拿到特别流畅的回答，就会锚在那个高点上，拿这个高点去评判后面的回答；对照组用户的锚点更低，对同等质量的输出评价更正面。在 LLM 实验里，即使加了思维链提示和明确的反锚定指令，锚定偏差在统计上依然显著。[^10] 不要用被试内设计（within-subjects），让用户先后看到旧输出和新输出。

## 诚实地读一个零结果

大约三分之一的实验会跑出不显著的结果。[^11] p 值高于 0.05 不代表没有效应，它只说明在零假设下数据与“无效应”相容。三种情形会给出同一个读数，而它们需要不同的应对：

1. **真零结果。** 处理确实没有提升。应对：接受，不上线。
2. **功效不足（underpowered）。** 实验的 N 不足以检测出处理实际产生的提升。应对：看置信区间。如果它同时包含你的目标 MDE 和 0，就是功效不足。延长实验或扩量。
3. **方差膨胀。** 处理有真实提升，但模型内方差在当前的 N 下把它盖住了。应对：上 CUPED、用配对分析，或者跑一个更大的实验。

置信区间是区分它们的那件工具。一个“无显著效果”配 CI [-0.5%, +4.5%]，与 4 个点的提升完全相容，你是功效不足。一个“无显著效果”配 CI [-0.2%, +0.3%]，则是强证据表明处理不超过 0.3 个百分点，零结果可信。Spotify 的 Confidence 团队把这个机制讲得很精确：计算器假设的是固定样本检验，而实际分析用的是始终有效置信序列（always-valid confidence sequence），这会让所需 N 被低估三分之一甚至更多，结果是计算器给出绿色对勾、分析给出不确定读数，而没有任何迹象表明两者从来就没对上过。[^12]

偷看（peeking）问题真实存在，而且很容易踩进去。在名义 alpha=5% 下对一个固定样本期的检验检查 20 次，假阳性率会飙到大约 40%。[^1] 解法是从一开始就用序贯检验（always-valid p 值、mSPRT 或 O'Brien-Fleming 边界），并在实验开始前把停止规则写进文档。相比固定样本检验，O'Brien-Fleming 边界只把最大所需 N 抬高几个百分点，而始终有效置信序列要拿到同样功效需要多约 50% 的观测。选你要的那个权衡。[^12]

零结果事后检查清单：

- 随机化是否正确（样本比例失配检查是否通过）？
- 指标是否在实验开始前就埋好，而不是事后补的？
- N 是否至少达到了计划目标？
- CI 是否把目标 MDE 排除在外？

如果四项都是“是”，这个零结果可信。任何一项是“否”，就去修埋点重跑，而不是重新解释结果。

## 什么时候离线评估就够了

Nextdoor 公开的案例（2023 年 12 月）是 AI 功能离线与线上分化最干净的数据点。他们把 ChatGPT 生成的邮件主题行与规则基线（帖子开头几个词）做 A/B 测试。原始 ChatGPT 的点击率只有基线的 56%，尽管生成的主题行“信息量更大”。[^13] 原因是：生成的行读起来像营销文案，损害了真实感。他们的解法是用历史 A/B 点击数据训练一个奖励模型，在 N 个 ChatGPT 候选里做挑选。最终结果：对比规则基线，会话数 +1%、WAU +0.4%、广告收入 +1%。

教训是：一个看起来合理的质量改进，并不蕴含更好的业务指标，而离线评估分不清这两者。

默认规则：离线评估把关，线上 A/B 决定上线。例外是流量确实撑不起一个有足够功效的 A/B 测试，这时退回离线评估，并配合显式的功效分析和诚实的不确定性估计，报告你能检测到的 MDE，而不是一个你其实无法主张的“无效应”。医疗、法律、金融这类安全要求高的领域在任何线上曝光前都维持更高的离线门槛，但最终裁决仍然由线上数据做。做对的团队把离线评估当回退过滤器，而不是上线标准。

## 参考文献

[^1]: Tian Pan, "Variance Eats the Experiment: Why A/B Power Math Breaks for LLM Features", April 2026. https://tianpan.co/blog/2026-04-27-ab-test-power-stochastic-features

[^2]: Haase et al., "Within-Model vs. Between-Prompt Variability in Large Language Models for Creative Tasks", arXiv:2601.21339, January 2026. https://arxiv.org/abs/2601.21339

[^3]: Nicholson, "Quantifying Non-Deterministic Drift in Large Language Models", arXiv:2601.19934, January 2026. https://arxiv.org/abs/2601.19934

[^4]: Anthropic, "A Statistical Approach to Model Evaluations", November 2024. https://anthropic.com/research/statistical-approach-to-model-evals

[^5]: Statsig, "Beyond Prompts: A Data-Driven Approach to LLM Optimization", March 2025. https://www.statsig.com/blog/beyond-prompts-llm-optimization

[^6]: Graham McNicoll, GrowthBook, "How to A/B Test AI Features", June 10, 2026. https://blog.growthbook.io/how-to-a-b-test-ai-a-practical-guide/

[^7]: Tian Pan, "A/B Testing AI Features When the Treatment Is Non-Deterministic", April 19, 2026. https://tianpan.co/blog/2026-04-19-ab-testing-llm-features-non-deterministic

[^8]: Airbnb Engineering, "Beyond A/B Test: Speeding Up Airbnb Search Ranking Experimentation through Interleaving", October 2022. https://medium.com/airbnb-engineering/beyond-a-b-test-speeding-up-airbnb-search-ranking-experimentation-through-interleaving-7087afa09c8e

[^9]: Chishti, Oyinloye, Li, "Test Before You Deploy: Governing Updates in the LLM Supply Chain", arXiv:2604.27789, April 2026. https://arxiv.org/html/2604.27789v1

[^10]: Tian Pan, "The AI A/B Test That Lied: Novelty, Carryover, and Anchoring Bias in LLM Experiments", May 7, 2026. https://tianpan.co/blog/2026-05-07-ai-ab-test-novelty-carryover-anchoring-bias

[^11]: Statsig, "Interpreting Non-Significant A/B Test Results", August 27, 2024. https://www.statsig.com/blog/interpreting-non-significant-ab-test-results

[^12]: Mårten Schultzberg, Spotify Confidence, "What Makes a Good Sample Size Calculator?", May 27, 2026. https://confidence.spotify.com/blog/good-sample-size-calculator

[^13]: Zeng et al., Nextdoor, "Let AI Entertain You: Increasing User Engagement with Generative AI and Rejection Sampling", arXiv:2312.12457, December 2023. https://arxiv.org/abs/2312.12457
