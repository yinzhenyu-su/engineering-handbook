---
title: "你的第一套评估集"
description: "一周之内搭起一套 50 条样本的评估集：用真实或足够逼真的输入，打二元的通过/不通过标签，放进 git 做版本管理，再从线上失败中不断长大。"
slug: 02-your-first-eval-set
part: 4
chapter: 2
difficulty: beginner
prerequisites:
  - 00-why-you-cant-ship-without-evals
  - 01-look-at-your-data
  - part-3-prompt-engineering/04-prompt-versioning
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 你的第一套评估集

产品经理问新 prompt 是不是更好。答案大致是“我在 playground 里试了十五个例子，看着更利索了”加一个耸肩。一周之后，模型 API 推了个小版本更新，还是那十五个例子，看着依然没问题，然后一条客户升级投诉落到 Slack 上，来自一个没人想到要去试的输入。团队是从用户嘴里知道这次回归的，不是从一个数字里。

这就是没有评估集就上线的感觉。解法不是买个平台、换个厂商，或者搞一场季度专项。就是一个大约 50 行的 JSONL 文件，签进 git，能在不到一分钟内回答一个问题：这次改动让系统变好了还是变差了？

起步集故意做得很小。50 行足够暴露多种多样的失败模式，又小到一名工程师加一名领域专家花两三个下午就能标完。[^1] 再往后，这套集合就像测试套件一样长大——靠吸收线上不断冒出来的 bug。

## 一条评估记录到底长什么样

先看实物。一行 JSONL，六个字段：

```json
{"id": "ex001",
 "input": "show me the slowest traces yesterday",
 "expected_label": "fail",
 "critique": "Query returns MAX(duration_ms) over time but doesn't group by trace.trace_id, so the user sees an aggregate line, not the actual slow traces.",
 "source": "production",
 "tags": ["feature:nlq", "scenario:aggregation"]}
```

六个字段里有五个不言自明。真正干活的是 `critique`。它是标注员用大白话写下的一句话理由，说明这条为什么判通过、那条为什么判失败。将来这句话会变成 LLM 裁判的 few-shot 上下文，也是第二个标注员用来核对自己会不会做出同样判断的依据。没有评语的标签，是一张无法审计的选票。[^2]

Honeycomb 团队的自然语言查询功能就是这么起步的。Phillip Carter 是懂那套查询语言的领域专家，他给每一个 fail 都写一行评语。把这些评语回灌给 LLM 裁判，迭代三轮之后，裁判与 Carter 的一致率超过 90%。[^2] 这些评语不是文档，是训练信号。

其他字段也各自挣到了位置。`source` 区分合成种子和真实线上 trace，这样你能看着这套集合慢慢成熟。`tags` 让你检查在功能、场景、用户画像上的覆盖度。`id` 让一行记录可以被回归报告引用。`expected_label` 是二元的——这就是下一个值得争论的点。

## 二元标签胜过 1 到 5 分

第一反应是分档打分。输出大概对，给 3 分。输出很棒，给 5 分。想法是：一个数字比一个是非题承载更多信息。实际恰恰相反，数字承载的更少。

两个标注员看同一条输出，在五分制上大约有一半的时候会不一致。Eugene Yan 报告说，LLM 标注任务上标注者之间的 Cohen's Kappa 只有 0.2 到 0.3，落在“fair”区间的低端；他还提到，人一疲劳，在一长批标注里会漏掉多达 50% 的缺陷。[^3] Hamel Husain 在三十多个落地项目里和领域专家合作过，说得更直白：“在一套 1 到 5 分的标尺上追一堆分数，往往是评估流程有问题的信号。”[^2] 他给的两条理由都很硬。一是人不知道 3 分和 4 分该怎么处理。二是“4 分算不算通过”这个阈值问题没有答案，于是每个利益相关方各自谈出一个不同的答案。

![两位标注员并排给同样的十条输出打标签，左侧二元通过/不通过一列高度一致，右侧 1 到 5 分一列频繁出现一两分的分歧](../../../ai-eng/part-4-eval-fundamentals/images/02-binary-vs-likert-agreement.png)
*分歧的形态本身就是论据：二元只逼出一个判定边界，李克特量表留了五个。*

二元标签胜出的更深一层原因：它逼出一个判定边界，而评估集存在的意义就是划定判定边界。一个通过/不通过的标签回答了部署唯一关心的问题：这条输出有没有达成想要的结果？3 分不能决定上不上线；只有阈值能决定上不上线，而这个阈值你最后总得选。Llama2 团队算过一笔类似的账：成对比较每个标注单元约 3.50 美元，写成评分的有监督数据每个单元约 25 美元。二元标签把认知负担和单价一起压了下去。[^4]

诚实的例外是：两个 prompt 变体都通过，而你需要挑出更好的那个。这时候成对比较就派上用场：给两条输出，问哪条更好，允许平局。[^3] 成对比较恰好在你需要的地方保住了序数信息——两个接近的候选之间——又不用逼标注员在几百个例子上校准一把五分尺。

> [!WARNING]
> **别在 bad/ok/good 三分制上妥协。** 它看起来像个折中立场，其实不是。你照样继承同样的校准问题（什么算“ok”而不是“good”？），却丢掉了序数分辨率，阈值问题还在后头追着你。Husain 明确警告过，早期阶段不要这么干。[^2] 如果真正的二元对你这个任务太粗，下一步是成对比较，不是缩短版的李克特量表。

## 怎么凑到五十条

50 这个数字不是样本量算出来的。它是大多数团队不再看到新失败模式的实战区间。[^1] 真正的停止条件是“我们连着标了十条，一条新东西都没有”，这通常落在 30 到 50 行之间。如果你的任务有很多子功能，预期要接近 100 或 200；到 200 条时，5% 的缺陷率给出的 95% 置信区间大约是 ±3 个百分点，够你做上/不上线的判断了。[^3]

种子来自三个来源，按优先级排：

- **真实线上 trace。** 只要有任何流量，这就是唯一重要的来源。拉最近一周的输入，按用户类型抽样，过一遍你当前的系统，然后给输出打标签。
- **合成输入，但跑真实系统。** 还没有流量时，用一个更小更弱的模型来生成输入。“弱生成器”这条规则很关键：前沿模型生成的输入太干净，弱一点的模型才会产出那种脏兮兮、有机的边缘情况，接近真实用户真正会敲的东西。[^3] 把这些合成输入过一遍你真正的系统，再给输出打标签。
- **手工构造的边缘情况。** 用来覆盖你知道存在但还没见过的场景（拒答、对抗输入、你设计师那轮 QA 漏掉的那种错别字）。这些最后加；它们补空缺，不负责给整套集合立骨架。

你要追的是三个维度上的覆盖：功能（系统能做什么）、场景（正常、边缘、对抗）、用户画像（是谁在用）。[^1] 覆盖检查就是一个 Counter 字典：

```python
import json
from collections import Counter

REQUIRED_FEATURES = {"summarization", "qa", "refusal"}
REQUIRED_SCENARIOS = {"edge_case", "normal", "adversarial"}

def coverage_report(eval_path: str) -> dict:
    features: Counter = Counter()
    scenarios: Counter = Counter()
    with open(eval_path) as f:
        for line in f:
            row = json.loads(line)
            for tag in row.get("tags", []):
                if tag.startswith("feature:"):
                    features[tag[len("feature:"):]] += 1
                elif tag.startswith("scenario:"):
                    scenarios[tag[len("scenario:"):]] += 1
    return {
        "missing_features": list(REQUIRED_FEATURES - set(features)),
        "missing_scenarios": list(REQUIRED_SCENARIOS - set(scenarios)),
        "coverage_ok": (
            not (REQUIRED_FEATURES - set(features))
            and not (REQUIRED_SCENARIOS - set(scenarios))
        ),
    }
```

把它挂成 JSONL 文件上的 pre-commit hook。`coverage_ok` 为假，就说明你的集合挤在容易的例子周围。另一个值得跑的检查：数一下 `expected_label == "fail"` 的行。如果 fail 不到 20%，你的集合太容易了。信号都住在 fail 里，一套几乎全是通过的集合训出来的裁判，会学会把输出一股脑放行。[^3]

## 像代码一样管版本

评估集住在 git 里。一个 JSONL 文件，一行一条样本，和它所评判的代码一起原子提交。一万行以内，版本管理的全部故事就这些。超过这个量，或者评估里包含图片、音频这类二进制资产，就换 DVC：git 里只留一个小的 `.dvc` 指针文件，实际数据按内容哈希存到 S3 或 GCS。[^5]

版本管理为什么重要，第一次 debug 回归的时候就明白了。一个 pull request 改了 prompt，一次评估跑出来说分数从 0.82 掉到 0.79，你需要知道是 prompt 变差了，还是上周四有人往集合里塞了三个难例子。如果数据集没有一个跟这次运行绑定的不可变版本，这个问题无解。有了它，你 checkout 上一个 commit，重跑，从一个月前用的同一个数据集上把那个数字读出来。[^6]

纪律很小，但没得商量：数据集改动和代码/prompt 改动分成两个 commit。一个 PR 可以同时动两边，但 commit 要拆开，这样无论沿哪个维度 bisect 都还能用。为部署决策跑评估时给 commit 打个 tag，就像你打 release 一样。

下面这个 schema 够用了；忍住往里面加“以后可能有用”的字段的冲动。评语优先，来源留痕，标签自由：

```python
from dataclasses import dataclass, asdict
from typing import Literal
import json

@dataclass
class EvalRow:
    id: str
    input: str
    expected_label: Literal["pass", "fail"]
    critique: str
    source: Literal["synthetic", "production"]
    created: str
    tags: list

def append_row(path: str, row: EvalRow) -> None:
    with open(path, "a") as f:
        f.write(json.dumps(asdict(row)) + "\n")
```

Braintrust、LangSmith、Langfuse 这类工具提供托管的数据集版本管理，还带一个把 trace 提升成评估行的 UI。[^6] 等集合变大，或者评审人不是工程师了，值得采用。对“一名工程师加一名领域专家”的起步阶段，JSONL 放 git 里更快、免费，而且在你手上任何一个 code review 工具里都能审。

> [!WARNING]
> **评估记录是存档，不是草稿纸。** 把 JSONL 文件当成可以随手就地改的东西（“就把第 12 行的期望标签改了一下，看着不对”），会毁掉可复现性。一行一旦进来，内容就冻结了。重打标签要作为一次新 commit，消息写清楚“relabel batch from criteria audit”，旧标签在 git 历史里仍然可读。schema 变更走迁移，不靠一条 sed 命令。[^7]

## 让线上失败把它喂大

第一天的起步集，代表的是你想象中的失败分布。三个月后，用户已经用你没想到的方式把系统推到了别处，一套停在第一周的集合在考一张错的卷子。集合必须长大，而唯一值得的养料是真实的线上失败。[^6]

这个闭环有四步：

1. **在线上记录每个请求和响应。** trace ID、输入、输出、模型版本、prompt 版本、时间戳。
2. **用你在离线用的同一个裁判给每条 trace 在线打分。** 一个二元通过/不通过信号，写回 trace。
3. **给低分的那批做分诊。** 有人每周花 30 分钟读失败样本，确认裁判确实没有看错，再挑出值得加进集合的。
4. **把确认的失败作为新行提交**，`source: "production"`，配一句新的评语。

分诊这一步是大多数团队把闭环做死的地方。没有轮值的人，第一周跑，第二到第十二周不跑，等有人注意到的时候，集合已经两个月没进过新行了。解法一点也不隆重：把每周分诊指派到人，把“每个 sprint 新增多少条样本”当成指标，定个像“每个 sprint 五条新线上失败”的目标。

有一个选择偏差值得直接点出来。只用那些留下来跟你交互了第二次的用户来建集合，会漏掉把其他用户赶走的那些失败模式。评估集上的通过率在爬，仪表盘上的留存率在掉。缓解办法是显式地追踪退出交互。如果用户点了一次 AI 功能再也没回来，那次交互就是集合的候选，哪怕没有人投诉。[^8]

Hamel Husain 在多个落地项目里用的迭代套路：前两轮标注用随机采样，之后转向定向搜失败。[^2] 找到一种已确认的错误类型，在日志里搜更多同样形状的例子，成批加进来。底下再铺一层薄薄的随机采样，这样你还能继续捞到那些你还不知道要去搜的新失败模式。

线上流量跑满 60 天之后，看一眼集合里 `source` 的分布。如果 production 占比低于 20%，说明这个闭环没在转。这个数字是成熟度仪表：健康的起步集以合成为主，健康的成熟集以线上为主。

## 标准漂移是真的，评语是防线

评估集最难的地方在于，标注员的标准会随着标注而变。Shankar 等人（2024）证明这几乎是普遍现象：即使是先给输出打了分的人，也会在后续打分中修正自己的标准，回头改掉先前给出的分数。[^9] 你第一周说的“通过”不是第八周说的“通过”。整体趋势会变得很滑。

防线就是评语字段。标签只是一个 token；评语是产出它的那段推理。等标准稳定下来，你回头读早期评语，并用现行标准重新过一遍那些标签。定期挑集合早期的一个 20 行子集，用今天的标准重标一遍；如果翻转率超过 15%，就安排一次重标。评语让这件事很便宜，因为标注员不用从头重新判断，只需检查旧的推理是否仍然站得住。

这里也接上了通往 LLM 裁判的那条线。你在这里写的评语，就是 LLM 裁判以后要用的 few-shot 上下文；这里的二元标签，就是那个裁判被训练去对上的目标。裁判不可能比它学习的人类标签更一致。评语越早写清楚，下游的一切越便宜。[^2]

至于可观测性平台如何让生产日志这一步真正跑起来、放大到架构尺度长什么样，HLD Part 9 的[可观测性平台](../../hld/part-8-case-studies/52-observability-platform.md)给出了白板视角。

## 参考文献

[^1]: Hamel Husain, "Using LLM-as-a-Judge For Evaluation: A Complete Guide," hamel.dev, 2024 年 10 月 29 日. https://hamel.dev/blog/posts/llm-judge/
[^2]: Hamel Husain, "Using LLM-as-a-Judge For Evaluation: A Complete Guide," hamel.dev, 2024 年 10 月 29 日（Honeycomb 案例、用评语做影子校验的流程、为二元标签辩护）. https://hamel.dev/blog/posts/llm-judge/
[^3]: Eugene Yan, "Product Evals in Three Simple Steps," eugeneyan.kit.com, 2024 年 12 月. https://eugeneyan.kit.com/posts/product-evals-in-three-simple-steps
[^4]: Eugene Yan, Bryan Bischof, Charles Frye, Hamel Husain, Jason Liu, and Shreya Shankar, "What We've Learned From A Year of Building with LLMs," applied-llms.org, 2024 年 6 月 8 日（第 1.4.5 节；Llama2 成本数据来自 Thomas Scialom）. https://applied-llms.org/
[^5]: DVC（iterative.ai）, "Versioning Data and Models," dvc.org, 2026 年 6 月访问. https://dvc.org/doc/use-cases/versioning-data-and-models
[^6]: Ankur Goyal, "Eval feedback loops," Braintrust blog, 2024 年 4 月 17 日. https://www.braintrust.dev/blog/eval-feedback-loops
[^7]: Tian Pan, "Why a Prompt Schema Change Wrecks 800 Test Cases," TianPan.co, 2026 年 5 月 2 日. https://tianpan.co/blog/2026-05-02-eval-migration-tax-prompt-schema-changes
[^8]: Tian Pan, "Why Your Test Set Goes Blind to the Failures That Drove Users Away," TianPan.co, 2026 年 5 月 10 日. https://tianpan.co/blog/2026-05-10-eval-selection-bias-test-set-blind-to-churn-failures
[^9]: Shreya Shankar, J.D. Zamfirescu-Pereira, Bjorn Hartmann, Aditya G. Parameswaran, and Ian Arawjo, "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences," arXiv:2404.12272, 2024 年 4 月 18 日（ACM UIST 2024）. https://arxiv.org/abs/2404.12272
