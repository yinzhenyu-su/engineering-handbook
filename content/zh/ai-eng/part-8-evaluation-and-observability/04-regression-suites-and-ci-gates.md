---
title: "回归套件与 CI 门禁"
description: "把评测接进 CI 当成真的合并门禁：两层黄金集、以差分阈值取代绝对地板，以及如何让抖动的概率性测试仍然可信。"
slug: 04-regression-suites-and-ci-gates
part: 8
chapter: 4
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/02-your-first-eval-set
  - part-4-eval-fundamentals/03-assertions-and-unit-tests
  - 01-llm-as-judge
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 回归套件与 CI 门禁

有个团队（名字我就不提了）上线了一道让他们很自豪的 prompt CI 门禁：200 条黄金集上通过率必须过 85%，否则合并按钮保持灰色。六周之后，平均质量从 0.93 漂到 0.87。每个 PR 都过了线。每个 PR 都合了。没有哪一个改动该为这次回退负责，因为没有任何一个改动破了规则。[^1]

这就是本章要讲的失效模式。一道朴素的评测门禁不是质量控制，而是质量表演。修法分三块，必须一起落地：一个规模足够真正检出回退（regression）的黄金集（golden set）；一道跟自己基线比、而不是跟静态地板比的门禁；以及对天生带概率性的输出保留足够的统计上的克制。

## 为什么显而易见的阈值会失效

85% 绝对阈值门禁只量一件事：瞬间的位置是否在某条线之上。它不量方向。一个把分数从 0.92 压到 0.86 的 PR 会绿灯上线；一个把分数从 0.80 抬到 0.84 的 PR 反而过不了同一道门禁。团队学到的规则是“过线就发”，而这条规则跟“维持或提升质量”是两回事。

![LLM 评测通过率在六周内从 93% 下滑到 87% 的折线图，每个 PR 都因为单次得分高于 85% 的绝对阈值线而合并，珊瑚色箭头标出未被察觉的漂移](../../../ai-eng/part-8-evaluation-and-observability/images/04-drift-failure-mode.png)
*六周“通过”的合并，丢掉六个点的质量。绝对阈值门禁量的是位置，不是方向；改动的主人是所有人，也是没有人。*

诚实的门禁是对比式的（differential）。你拿 PR 分支在同一黄金集上的分数，跟基线分支在同一黄金集上的分数比，只有当增量（delta）为负、而且大到算得上真信号而不是噪声时，才让构建失败。后半句比听起来更要紧，因为概率性测试的噪声地板比工程师预期的高得多。

## 确定性在哪里崩塌（BF16 的意外）

多数工程师以为 `temperature=0` 加上固定 seed 就能得到确定性测试。并不行，原因在硬件，不在模型。BF16 浮点运算不满足结合律：`(a+b)+c` 与 `a+(b+c)` 可能在最后一位上有差别。改动 GPU 数量、batch size 或 GPU 型号，累加顺序就变了。当 top-1 与 top-2 的 logits 很接近时，那最后一位就会把 token 选择翻过去。解码出来的文本分叉，评测结果跟着翻。

Yuan 等人在 DeepSeek-R1-Distill-Qwen-7B 上、AIME'24 的贪婪解码（temperature=0）条件下，跨 12 种硬件配置直接测了这一点。BF16 精度下各配置之间准确率的标准差是 9.15%；在 MATH500 上，99.6% 的样例在不同配置间产出了不同输出。把同一模型换成 FP32 推理，Std@Acc 降到 0%，MATH500 上的分叉率降到 19.9%。[^2] 对 CI 的启示很直接：如果你的服务商用 BF16 推理，“我设了 temperature 为零”不是防抖动策略，是猜。

OpenAI 自 2023 年 11 月起提供的 `seed` 参数，文档写明是尽力而为、不保证：后端一变，`system_fingerprint` 字段就变，可复现性随之破掉。[^3] 另一项跨五个模型的研究报告，temperature=0 时不同轮次之间的准确率波动最高达 15%。[^4] 把 seed 当成一层防御，别当成答案。

那就只剩两个真工具：把每条样例跑多遍再聚合；把阈值放宽到一个置信带，容纳你杀不掉的噪声。

## 多采样聚合与置信区间

最小可用的防抖动手段，是把每条黄金样例跑 N 次再聚合。对 CI 来说，N=5 加多数投票（每个样本二值通过/失败）能把虚假失败砍掉一大截，按常见评测模型的价格，每次构建多花约 $0.15 到 $0.50。夜间套件不那么在乎成本，N=16 或更高也合理。Song 等人建议任何安全相关的评测至少取三个样本；单次采样与多样本真值的一致率只有 92.4% 到 97.7%，具体看配置。[^5]

第二个工具是统计层面的：门禁得知道噪声到底多大。朴素的中心极限定理置信区间 `SE = sqrt(p*(1-p)/N)` 在小的黄金集上会骗人。Bowyer 等人（ICML 2025）证明，N 小于几百时，CLT 区间系统性地低估不确定性，画出来的误差棒看着很窄，统计上是错的。[^6] N=50、通过率 80% 时，真实的 Wilson 95% 区间是 [0.67, 0.89]，一个 22 个点的窗口。你在 50 条样例上把基线 0.84 跟 PR 的 0.80 一比就宣布回退，那是把噪声当信号读。

改用 Wilson 得分区间或贝叶斯 Beta-Bernoulli 分析。两者在 `scipy.stats.proportion_confint(..., method='wilson')` 里都只有一次函数调用之遥。门禁于是变成：只有当 PR 置信区间的上界掉到你的地板以下，或与基线的配对比较显示出统计上显著的负向增量时，才失败。

## 一份真正扛得住的准入门禁脚本

下面这份是最小可用脚本。它读一个结果文件（这个形状对应 `promptfoo` 的 `results.json`，不过任何通过/失败计数都能用），算 Wilson 置信区间，只在 CI 的上界掉到带容差的地板以下时才让构建失败。

```python
import json
import sys
from scipy.stats import proportion_confint


def eval_gate(
    results_path: str,
    threshold: float = 0.80,
    tolerance: float = 0.05,
) -> int:
    """Wilson-CI gate: fail only when upper bound is below the noise floor."""
    with open(results_path) as f:
        data = json.load(f)

    stats = data["results"]["stats"]
    successes = stats["successes"]
    total = successes + stats["failures"]
    if total == 0:
        print("No results found", file=sys.stderr)
        return 1

    lo, hi = proportion_confint(successes, total, alpha=0.05, method="wilson")
    pass_rate = successes / total
    floor = threshold - tolerance

    print(f"Pass rate: {pass_rate:.2%}  Wilson 95% CI: [{lo:.2%}, {hi:.2%}]")
    print(f"Floor: {floor:.2%} (threshold {threshold:.2%}, tolerance {tolerance:.2%})")

    if hi < floor:
        print(f"GATE FAILED: upper CI {hi:.2%} below floor {floor:.2%}")
        return 1
    print("GATE PASSED")
    return 0


if __name__ == "__main__":
    sys.exit(eval_gate(sys.argv[1]))
```

真正干活的是 `if hi < floor` 那一行。50 条样例跑到 79% 通过率，Wilson 上界大约 88%；这证据不足以拦住一个合并。200 条样例跑到 70%，上界大约 76%；这是真失败，门禁开枪。

把它接进 GitHub Actions，配一个路径过滤，让评测只在 prompt 或模型配置真的变了时才跑：

```yaml
name: LLM Eval
on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'promptfooconfig.yaml'
      - 'evals/**'
jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22' }
      - name: Run eval
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: npx promptfoo@latest eval -c promptfooconfig.yaml -o results.json
      - run: pip install scipy
      - run: python scripts/eval_gate.py results.json
```

这个路径过滤，就是每月 20 美元 CI 账单和每天 20 美元之间的差别。每次改 CSS 都跑一遍评测，是最容易让门禁贵到有人干脆把它关掉的做法。

## 两层，而不是一层

单个黄金集不可能既快到每个 PR 都跑，又全到能抓住罕见的回退。所以跑两个：

- **快层（每个动到 prompt 的 PR）。** 50 到 200 条样例，以确定性[断言](../part-4-eval-fundamentals/03-assertions-and-unit-tests.md)为主，断言够不着的地方用便宜的评审模型。目标：三分钟以内、每次构建 1 美元以内，拦住合并按钮。
- **夜间层（全量套件）。** 500 到 2,000 条样例，含 LLM-as-judge 打分、更长的轨迹、更硬的边界情况。结果发到看板，出现回退就开 issue，但不拦住触发它的那个 PR。

低于 50 条样例，Wilson 区间宽到抓不住五个点的回退，快层就失去意义。超过约 200 条，构建时间和成本会把工程师推向“重跑到绿为止”的行为，那比没有门禁更糟。夜间层存在的意义，就是收下所有不符合这些约束的东西。

两层之间有个好用的反馈回路。夜间运行抓到快层漏掉的真实回退时，那条样例被提进快集。生产故障经你的 tracing 与反馈系统冒出来时也一样：每个定位清楚的 bug 变成黄金集里一行带标注的数据，配一个能抓住它那个具体模式的打分器。Braintrust 把这套东西明确写成回归测试流水线（regression-test pipeline）；同一个回路对任何平台、或者仓库里一个扁平 JSON 文件都成立。[^7]

## 一段话讲清黄金集的整理

黄金集不是生产流量的随机采样，而是有意的横切面：把真实 trace 聚成 5 到 15 个行为类别（拒答、结构化抽取、多跳推理、接近边界的情形、业务关键路径），每类挑 5 到 20 条代表样例，能写确定性断言的就写，只有断言量不出来的性质才加 LLM-as-judge 打分。文件跟应用代码一起放在版本控制里，在有意改变行为的那个 PR 里同步升数据集版本。Hamel Husain 和 Shreya Shankar 的 evals 课程深入讲了整理这门手艺；part-4 那篇[你的第一个评测集](../part-4-eval-fundamentals/02-your-first-eval-set.md)走的是最小版本。[^8]

一个不那么显眼的风险：至少留一片私有样例，别放进公开仓库。基础模型会在抓取的 GitHub 上重训。一套在公开仓库里放了两年的黄金集，有不小的概率漏进下一个模型的预训练数据，你的分数会上涨，产品却没变好。

## 隔离不稳定的测试，而不是容忍它们

就算有了多采样投票和 Wilson 门禁，仍有少数黄金样例会不可预测地抖：标准答案本身模糊、LLM 评审跟自己意见不一、某个边界情况里模型每次给出正确但措辞不同的答案。按样例跟踪抖动率。任何在代码没动的情况下失败率超过 10% 的样例，先挪到夜间套件去，同时诊断原因；等连续两周低于 5%，再提回 PR 门禁。

这件事要紧的原因是文化，不是统计。两个抖动测试就足够教会一个团队：失败检查的按钮意思是“点重跑”，不是“读 diff”。这个习惯一旦成形，门禁又变成表演，你就回到了本章开头的地方。

放到架构尺度上，[评测与可观测性流水线](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md)讲了回归套件的信号怎么喂进生产告警、采样策略，以及夜间结果最终落到的那些长期看板。

## 参考文献

[^1]: tianpan.co, "Eval Differential as Branch Protection," engineering blog, April 28, 2026, https://tianpan.co/blog/2026-04-28-eval-differential-branch-protection-score-diff-not-floor
[^2]: Jiayi Yuan, Hao Li, et al., "Understanding and Mitigating Numerical Sources of Nondeterminism in LLM Inference," arXiv 2506.09501, May 2026, https://arxiv.org/abs/2506.09501
[^3]: OpenAI, "Reproducible outputs," OpenAI API Reference, accessed June 2026, https://platform.openai.com/docs/guides/text-generation#reproducible-outputs
[^4]: Label Studio, "How to handle non-determinism in agent evaluation," LabelStudio Learning Center, June 2026, https://labelstud.io/learningcenter/how-to-handle-non-determinism-in-agent-evaluation
[^5]: Yifan Song, Guoyin Wang, Sujian Li, Bill Yuchen Lin, "The Good, The Bad, and The Greedy: Evaluation of LLMs Should Not Ignore Non-Determinism," arXiv 2407.10457, July 2024, https://arxiv.org/abs/2407.10457
[^6]: Sam Bowyer, Laurence Aitchison, Desi R. Ivanova, "Position: Don't use the CLT in LLM evals with fewer than a few hundred datapoints," ICML 2025, arXiv 2503.01747, https://arxiv.org/abs/2503.01747
[^7]: Braintrust, "How to turn LLM production failures into regression tests," Braintrust engineering blog, May 2026, https://www.braintrust.dev/articles/turn-llm-production-failures-into-regression-tests
[^8]: Hamel Husain and Shreya Shankar, "AI Evals FAQ: What are LLM Evals?", AI Evals Course materials, 2025-2026, https://hamel.dev/blog/posts/evals-faq/what-are-llm-evals.html
