---
title: "为什么没有评测就上不了线"
description: "凭感觉开发的失效模式、把评测当作 AI 工程师的 CI，以及让你脱离打地鼠循环的成熟度阶梯。"
slug: 00-why-you-cant-ship-without-evals
part: 4
chapter: 0
difficulty: beginner
prerequisites:
  - part-0-start-here/02-thinking-in-probabilistic-systems
  - part-2-working-with-model-apis/00-calling-models
  - part-3-prompt-engineering/00-anatomy-of-a-production-prompt
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 为什么没有评测就上不了线

2024 年 3 月，Hamel Husain 发布了一份关于 Lucy 的事后复盘。Lucy 是 Rechat 为房地产从业者做的对话式 AI 助手，在 demo 里表现良好。到了生产环境，团队撞了墙。修好一种失效模式，另一种就冒出来；往系统提示词里加指令，提示词就膨胀成没人能完全看懂的东西。Husain 用的说法是“一场打地鼠游戏”。而诊断其实更简单：“除了凭感觉（vibe checks），我们对 AI 系统在各类任务上的效果几乎看不见。”[^1]

这就是本章要讲的失效模式。它如今有了名字，“凭感觉开发”（vibes-driven development），是 LLM 产品在 demo 之后停滞的最常见原因。解药同样有名字，而且直接借自常规软件工程：写测试，每次改动都跑，失败就挡住发布。我们把这些测试叫做*评测*（eval），Part 4 剩下的部分讲怎么把它们建起来。本章讲的是为什么你不能往后拖。

## 打地鼠式的循环是结构问题，不是能力问题

Rechat 团队并不马虎。他们撞上的是每一个没有评测套件的 LLM 产品的默认状态。四种机制在这里叠加，其中任何一种都足以让你上线一个没人发现的回归。

第一种是 **demo 选择偏差**。开发者测的是自己写的提示词，再加上用户在他们的新手引导阶段展示过的那几条。这些提示词跟生产分布完全不是一回事。demo 能用，是因为它被挑出来就是要能用的。

第二种是**静默的模型漂移**。API 供应商会在不改 API 签名的情况下更新端点背后的模型。OpenAI 的 gpt-3.5-turbo 在 2023 年至少更新过两次，客户端完全看不到信号。Wanqin Ma 和同事测了十组这样的更新，发现 58.8% 的“提示词 + 模型”组合在更新后准确率下降，其中 70.2% 的下降超过 5 个百分点（截至 2024 年 2 月）。[^2] 更让人警醒的细节：63.8% 的单个回归，发生在旧模型高度自信的输入上。断裂点不在决策边界，而是散布在整个分布里。没有回归套件，每一次断裂都无声上线。

第三种是**打地鼠式的提示词工程**。LLM 没有互相隔离的指令通道。你在系统提示词里为修掉一种失效模式加的每一行，都会重塑其它所有地方的输出分布。没有度量，你只能猜这个修复是整体变好了，还是只是把失败挪到了你没在看的地方。Rechat 的提示词之所以变得“又长又笨重，试图覆盖大量边界情况”，正是因为没人能确认这些追加是净收益。[^1]

第四种是**没有回归底线**。没有可回滚的基线，没有可辩护的数字，没有信号告诉你今天的发布比上周好还是坏。团队是从用户嘴里知道东西坏了的。

Shreya Shankar 在 CSCW 2024 发表了一份关于 18 位跑生产流水线的 ML 工程师的民族志研究，把结果凝成一句话，还成了论文标题：“We have no idea how models will behave in production until production.” 意思是在生产之前，我们根本不知道模型在生产里会怎么表现。[^3] 评测存在的意义，就是把系统从这个状态里捞出来。

## 评测就是 AI 工程师的 CI

常规软件里的持续集成做三件事：维护一套定义正确行为的测试套件，每次改动都跑这套套件，拦住失败的合并。eval-as-CI 模式把同样的形状搬到 LLM 系统上。**评测套件**（eval suite）是一组精选的“输入 + 期望行为”对，外加评分逻辑。任何触及提示词、模型版本、检索组件或流水线开关的 PR 都会跑它。评测失败就挡住部署，跟单元测试失败一样。

Anthropic 的工程团队在 2026 年 1 月把这种类比说得更直白：自动化评测“可以在每次提交时运行”，是“抵挡质量问题的第一道防线”。有可用评测套件的团队几天内就能评估并采用一个新模型；没有的团队每升一次级，都要面对几周的手工测试。[^4]

![一张双栏示意图。左边标着 vibes-driven，是一条闭环：Ship to users 指向 User complaints，指向 Fix one thing，指向 Break another thing，箭头绕回 User complaints。右边标着 eval-driven，是一条开放的有向循环：Code change 指向 Eval suite CI，指向 Green or Red gate，指向 Production monitor，再反馈 New test cases 回到评测套件；右侧一支珊瑚色箭头标出“失败变成测试”的反馈回路](../../../ai-eng/part-4-eval-fundamentals/images/00-vibes-vs-evals.png)
*凭感觉驱动的那一侧是个没有出口的环；评测驱动的那一侧是一个有向循环，每一次生产失败都会变成一个永久的测试用例。*

生产环境里的做法是把套件拆成两半。**回归评测**（regression evals）测的是已经能用的行为；通过率应该接近 100%，一旦下降就说明有东西坏了。**能力评测**（capability evals）测的是系统还不擅长的行为；起始通过率很低，存在的目的是跟踪改进。当某个能力评测爬到高通过率，就把它提拔进回归套件，让这份收益变成新的底线。[^4]

最简单的 CI 闸门就是一个脚本：加载 JSONL 测试集，跑过你的模型，通过率低于阈值就以非零码退出。要让 CI 判定一次构建失败，有这些就够了：

```python
import json
import sys
from typing import Callable

def regression_gate(
    dataset_path: str,
    eval_fn: Callable[[dict], bool],
    threshold: float = 0.95,
) -> None:
    with open(dataset_path) as f:
        cases = json.load(f)

    passed = sum(1 for c in cases if eval_fn(c))
    rate = passed / len(cases)
    print(f"Regression: {passed}/{len(cases)} passed ({rate:.1%})")

    if rate < threshold:
        print(f"FAIL: {rate:.1%} below threshold {threshold:.1%}")
        sys.exit(1)
    print("PASS: regression suite within bounds.")
```

95% 这个阈值故意低于 100%，因为 temperature 大于零时的 LLM 调用不是确定性的；阈值本身是个产品决策。真正让它成为 CI 的是非零退出码：GitHub Actions、GitLab 流水线以及其它所有 runner 都会把它当成失败的步骤，拦住合并。

## 评测成熟度阶梯

你不会在第一天就把整套系统建起来，也不该这么试。这个进程分几级，顺序很重要，因为每一级都靠下面那一级撑着。

**第一级：trace 日志。** 在任何评测成为可能之前，你得能重放发生过的事。记录每一次 LLM 调用：时间戳、钉住的模型版本、完整提示词、响应、延迟、所有工具调用及其结果、哈希后的用户 id、会话 id。如果你没法重建模型在某条用户投诉里收到了什么、产出了什么，你就没法给它打分，上面再套什么评测框架都救不了。Husain 在 Rechat 打法里的第一个具体动作，就是在 trace 存储之上搭了一个 Shiny 标注工具；后面的一切都是从那上面长出来的。[^1]

**第二级：基于断言的单元测试。** 便宜、快、确定。正则检查、schema 校验、结构性断言，比如“响应里永远不出现 UUID”或者“JSON 能解析”。这些跑起来只要毫秒，成本为零，能抓住最吵的那一类失败。Rechat 跑着“几百条”这样的测试。[^1] 起步 5 到 20 条就够；生产 trace 里冒出新的失效模式，你就往上加。

```python
import re
from dataclasses import dataclass
from typing import Callable

@dataclass
class EvalCase:
    input_text: str
    assertions: list[Callable[[str], bool]]

UUID_PATTERN = re.compile(
    r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}",
    re.IGNORECASE,
)

def no_uuid_in_response(response: str) -> bool:
    return len(UUID_PATTERN.findall(response)) == 0

def run_assertions(cases: list[EvalCase], model_fn: Callable[[str], str]) -> float:
    passed = 0
    for case in cases:
        response = model_fn(case.input_text)
        if all(a(response) for a in case.assertions):
            passed += 1
    return passed / len(cases)
```

UUID 这条断言是 Rechat 的经典例子：Lucy 会把内部房源 ID 泄到面向用户的回复里，一行正则就把这个 bug 的每一次回归永久挡住了。[^1]

**第三级：LLM 当裁判（LLM-as-judge）。** 很多失败不是结构性的。“回复很无礼”、“答案编造了一个日期”、“语气滑向了营销文案”，这些没法用正则抓。你把回复交给一个更强的模型，用一个提示词要它给出二元判定和一句点评。用二分类标签，不要用李克特量表；二元标签逼你把标准想清楚，标注者之间的一致性也更锐利。[^5] 你至少需要 100 个人工标注的样本才能标定裁判，因为 LLM 裁判有偏差（偏好更长的回复、更流畅的文字，以及跟自己风格相似的输出）。如果裁判相对你的人工标签的真阳率和真阴率掉到大约 0.80 以下，那问题就出在裁判身上，你该改它的提示词或者换裁判模型。

**第四级：生产环境的 A/B 测试。** 真实用户、真实结果、统计显著性。指标是行为层面的：任务完成率、留存、工单率。这是唯一能告诉你质量上的胜利有没有转成用户层面胜利的一级，也是最慢、最贵的一级：每个实验几天到几周，而且只有流量大得能盖过噪声时才有意义。[^4]

诱惑在于跳级，尤其是那些先听说 LLM-as-judge、后听说错误分析的团队。别跳。在没对真实 trace 做过错误分析之前造出来的裁判，往往在给错的东西打分；在回归套件还没变绿之前跑的 A/B 测试，只是在测噪声。Husain 教过 2,000 多名工程师和 PM，他给出的现场数字很直白：AI 产品开发时间里的 60% 到 80% 应该花在错误分析和评测上（截至 2026 年 1 月）。[^5] 如果你花得比这少，那你就是在发布感觉。

## 这套纪律要在第一个用户之前建立

人们最常输掉的论点——包括上面这些全都懂的团队——是时机。“评测我们后面再加，等 demo 能用了再说。” 后面就已经晚了。等到你有足够多的生产失败来知道该测什么，你已经烧掉了那些早期用户对你的信任，而他们正是撞上这些失败的人；而且你手上没有回归发生之前的基线可比。Anthropic 自己的 Claude Code 团队就是在“凭感觉加用户反馈”的模式上线之后补做评测的，他们明确把这条路描述成更难的那条；他们走到了，但代价比第一天就带评测起步的团队高。[^4]

最小可行门槛很小。20 到 50 个取自真实预期失败的样本，加 5 到 10 条基于断言的单元测试，再加一个在每次提示词改动时跑它们的 CI 步骤，就够你把一个功能发给一小批真实用户。[^4] 一个下午就能搭完。你唯一不能做的是继续往后拖，因为静默更新的问题不会等你的路线图。下次供应商轮换端点背后的模型，你只有两种知道的方式：几分钟内从 CI 知道，或者几周后从用户那里知道。这套纪律决定的是哪一种。

放到架构尺度上，[AI 系统的可观测性](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md) 讲的是本章假定你已经掌握的 trace 埋点白板视角。每一级具体怎么建，在 Part 4 剩下的部分：[先看你的数据](./01-look-at-your-data.md) 讲怎么确定该测什么，[你的第一个评测集](./02-your-first-eval-set.md) 讲标注集的纪律，[断言与单元测试](./03-assertions-and-unit-tests.md) 讲结构层，[LLM 当裁判](../../../ai-eng/part-8-evaluation-and-observability/01-llm-as-judge.md) 讲语义层。

## 参考文献

[^1]: Hamel Husain, "Your AI Product Needs Evals," Hamel's Substack, March 29, 2024. https://hamelhusain.substack.com/p/evals
[^2]: Wanqin Ma, Chenyang Yang, Christian Kastner, "(Why) Is My Prompt Getting Worse? Rethinking Regression Testing for Evolving LLM APIs," Conference on AI Engineering (CAIN 2024), April 2024. arXiv:2311.11123. https://arxiv.org/abs/2311.11123
[^3]: Shreya Shankar, Rolando Garcia, Joseph M. Hellerstein, Aditya G. Parameswaran, "'We Have No Idea How Models will Behave in Production until Production': How Engineers Operationalize Machine Learning," ACM CSCW 2024. https://arxiv.org/abs/2403.16795
[^4]: Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares, Jiri De Jonghe, "Demystifying Evals for AI Agents," Anthropic Engineering Blog, January 9, 2026. https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
[^5]: Hamel Husain and Shreya Shankar, "LLM Evals: Everything You Need to Know," hamel.dev, January 15, 2026. https://hamel.dev/blog/posts/evals-faq
