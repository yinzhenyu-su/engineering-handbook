---
title: "裁判的失败模式"
description: "LLM-as-judge 流水线里的位置偏差、长度偏差、自我偏好与风格偏差，连带实测数字和生产里真正管用的缓解手段。"
slug: 02-judge-failure-modes
part: 8
chapter: 2
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
  - 01-llm-as-judge
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 裁判的失败模式

2023 年 6 月，LMSYS 团队让 Claude-v1 当两两对比的裁判，用的是 MT-Bench 里 80 对质量相同的回答。同一个问题问两遍，只把哪条标成“A”、哪条标成“B”对调。Claude 有 76% 的情况推翻了上一次的判定。它挑的总是那条恰好排在前面。[^1]

你要是随随便便接上一个 [LLM-as-judge](./01-llm-as-judge.md) 就上线，交付出去的就是这种失败模式。裁判在打分，看板一片绿，而整个排名其实由提示词里的顺序决定。本章讲的就是造成这类破坏的四种有据可查的偏差，带上数字，也带上小到今天就能写进提示词里的缓解手段。

## 值得点名的四种偏差

有四类失真被测量得足够细，细到可以据此行动：

- **位置偏差（position bias）。** 裁判偏好落在某个固定位置上的回答（通常是第一条），与内容无关。
- **长度偏差（length bias）。** 裁判奖励那些零新增信息的灌水、重复和填充。
- **自我偏好（self-preference）。** 裁判偏爱自己所属模型家族的输出。
- **风格偏差（style bias）。** 内容完全相同，裁判偏好 markdown 排版（标题、列表、加粗）胜过纯文本。

这四者不能互换。长度偏差和风格偏差老被混为一谈，可修法并不一样：长度靠一句指令或者一次回归去压；风格要在裁判看到回答之前把格式归一化。它们的量级在不同代模型之间也不一致，而这一点你必须知道——所以正确的做法是拿你自己的裁判、在你自己的数据上逐项测量，而不是从 2023 年的论文里抄一个数。

![两个并排的面板，展示同一对 LLM 回答被评判两次，两个面板之间 A、B 的位置对调；两个面板里裁判都选了“A”，但 A 指向的是不同的实际回答，说明位置偏差造成了判定翻转](../../../ai-eng/part-8-evaluation-and-observability/images/02-position-flip.png)
*同一对回答，两版提示词，赢家相反。裁判读的不是内容，它锚在的是槽位。*

## 位置偏差

Zheng 等人在一个受控实验里测过这一项：GPT-3.5 生成的 80 对同质量回答，交给三个裁判各跑两遍，两遍位置对调。“一致”指两种顺序下的判定相同。[^1]

| 裁判（2023 年中） | 一致率 | 偏好第一条 | 偏好第二条 |
|---|---|---|---|
| Claude-v1 | 23.8% | 75.0% | 0.0% |
| GPT-3.5 | 46.2% | 50.0% | 1.2% |
| GPT-4 | 65.0% | 30.0% | 5.0% |

三点值得注意。第一，每个裁判都有偏好的一侧，而且偏好很强。第二，即便 GPT-4，把顺序对调之后，在同一对回答上也有 35% 的情况跟自己不一致。第三，两条回答质量越接近，偏差越大——而这恰恰是你最需要裁判判得对的时候。质量差距拉开时（GPT-3.5 对 LLaMA-13B），GPT-4 的一致率升到 98.8%；换成同等难度的主观写作题，掉到 42%。[^1]

2026 年的更新是实打实的，而且要紧。一项针对 2024—2025 年五个前沿模型（Gemini 2.5 Pro、Claude Sonnet 4、GPT-4o、Gemini 2.5 Flash、Llama 3.3-70B）的系统研究，在 400 个 MT-Bench 实例上测得五个模型的位置偏差分数都不高于 0.04，判定为可忽略。[^2] 也就是说，当前前沿裁判的位置偏差已经大幅缩小。但它没在更弱或更老的模型上消失，而且 2026 年那项研究用的是带质量差异的自然配对，不是偏差藏得最深的“同质量”合成配对。稳妥的规则：低于 GPT-4o 或 Claude Sonnet 4 的裁判，一律假定位置偏差不可忽略；信它之前先测。

### 修法：交换位置，不一致就判平局

保守的缓解方式是每次比较调两次：两种顺序各判一遍，只有两次判定一致才宣布胜负。

```python
def judge_with_swap(judge_fn, question: str,
                    response_a: str, response_b: str) -> str:
    """Pairwise judge with position-swap debiasing.
    Calls the judge twice with A/B swapped; ties when verdicts disagree.
    """
    verdict_ab = judge_fn(question, response_a, response_b)
    verdict_ba = judge_fn(question, response_b, response_a)

    # 交换后的那次调用里，模型说的 "A" 其实是我们的 B，所以要翻回来。
    flip = {"A": "B", "B": "A", "tie": "tie"}
    verdict_ba_normalized = flip.get(verdict_ba, "tie")

    if verdict_ab == verdict_ba_normalized:
        return verdict_ab
    return "tie"
```

成本是 2 倍 API 调用。再往上一步，是在提示词里加三个 few-shot 判定示例，原文论文里这招把 GPT-4 的一致率从 65.0% 提到 77.5%，提示词成本约 4 倍。[^1] 如果你已经在自己的数据上测过裁判的偏差、结果低于 5% 左右，或者候选之间的质量差距大到翻转顺序也改不了结论，那就可以跳过交换。最便宜的生产技巧是 Chatbot Arena 用的那招：每次请求随机分配 A/B。偏差在单次调用里仍然存在，但几千次比较下来就被平均掉了。[^1]

## 长度偏差

长度偏差最朴素的演示是重复清单攻击。拿一条本来就带编号列表的模型回答，把列表项换个说法追加到末尾（零新增信息），再让裁判拿注水版和原版比。裁判要是偏好注水版，攻击就成功了。

Zheng 在 2023 年用 23 条 MT-Bench 回答测出的结果：

| 裁判（2023 年中） | 攻击失败率 |
|---|---|
| Claude-v1 | 91.3% |
| GPT-3.5 | 91.3% |
| GPT-4 | 8.7% |

2023 年时 GPT-4 对这套清单攻击基本已经免疫。[^1] 但一般创意写作上的长度偏差比这个清单把戏宽得多。Saito 等人拿同一个生成器产出的 100 条长短不一的回答做了受控研究，报告 GPT-4 的冗长偏差（verbosity bias）分数是 0.328、GPT-3.5 是 0.428（正数表示裁判偏好冗长，取值区间 -1 到 +1）。[^3] 在人类偏好较短回答的样本上，GPT-4 跟人类不一致的情况比人类偏好较长回答时更多。这个偏差的方向恰好伤你的 RLHF 奖励信号：注水的回答会把生成器训得更爱注水。

再往下就是在规模上可被利用。Dubois 等人证明，只要提示同一个模型“尽可能啰嗦”，就能把它的 AlpacaEval 胜率从 22.9% 推到 64.3%。[^4] 同一个模型，同样的问题，字数多了，排名就变了。

2025 年之后的图景更有意思。那项把位置偏差压平的系统研究同时发现，当前的前沿裁判会主动惩罚灌水：在扩写对上（往一条完整回答里加填充），Claude Sonnet 4 得分 -0.76、GPT-4o 得分 -0.24，两个都偏好更短的那版。[^2] 但在截断对上（长的那版确实更完整），五个模型全都正确地偏好长版，准确率 0.92-1.00。现代裁判似乎学会了正确的规则：奖励完整，惩罚注水。三年前它们还不会。

### 修法：先改提示词，再上回归

对大多数生产流水线，最便宜的缓解是在裁判提示词里多写一句：

```python
LENGTH_NEUTRAL = (
    "Evaluate quality and accuracy only. "
    "Do NOT prefer a response simply because it is longer or shorter. "
    "If a longer response adds no information, treat it as equal or worse."
)
```

最后那句才是关键。“别偏好更长的”很容易被裁判读成“偏好更短的”，那你就翻进了另一种偏差。把失败模式点出来（注水，没有信息），才能让它在扩写对和截断对上都做对。

当你掌控一条 benchmark 的评分流水线时，更重的修法是长度受控回归：在一个比较池上拟合一个三项模型（系统、长度、指令难度），把长度项归零，还原出反事实胜率。AlpacaEval 以 length-controlled AlpacaEval 的名义发布的就是这个，它把与 Chatbot Arena 的 Spearman 相关系数从 0.94 提到 0.98。[^4] 这招只在 benchmark 规模下成立（你得有一池比较样本可拟合），单次调用的两两对比流水线里别用。

## 自我偏好

如果你拿 GPT-4 给一个本身跑在 GPT-4 上的系统当裁判，你就有麻烦了。Zheng 2023 年的论文里，GPT-4 给自己输出的胜率比人类裁判给的高出约 10%，Claude-v1 给自己高出约 25%。[^1]

Panickssery 等人在 2024 年找到了机制：裁判真的能认出自己写的东西。开箱状态下，GPT-4 区分自己的输出与另外两个 LLM 和人类的输出，两两准确率 73.5%。只在 500 个样本上微调之后，GPT-3.5 和 Llama-2 的自我识别率就超过 90%。而且在所有这些微调条件下都存在一条干净的线性相关：自我识别率上去，自我偏好跟着上去。在长度、元音数量或可读性上做的对照微调，没有产生同样效果。[^5] 自我识别是原因，不是巧合。

那篇论文里的摘要任务数字：

| 评测者 | 对比自己的输出 | 对比其他 LLM | 对比人类 |
|---|---|---|---|
| GPT-4 (XSUM) | - | 0.638 | 0.710 |
| GPT-4 (CNN/DM) | - | 0.619 | 0.715 |
| GPT-3.5 (XSUM) | - | 0.514 | 0.581 |

分数高于 0.5 表示评测者偏好自己的输出。在两个数据集上，GPT-4 大约 71% 的情况下偏好自己写的摘要胜过人类写的。[^5]

对 RLHF 和奖励建模流水线的含义，才是你该担心的部分。如果你用模型 M 生成训练数据、又用裁判 M 给它打分，你就是在训练步之间不断放大 M 的风格偏好。这个闭环会收敛到 M 本来就看好的自己，人类偏好被挤出画面。这直接接上 [红队测试](../../../ai-eng/part-9-security-and-governance/06-red-teaming.md) 里讲的安全问题：对手只要用跟你的裁判同一家族的模型，就能利用共享表征来操纵你的打分。

2026 年那项研究把“模型偏爱自己”这条简单规则搅浑了。在模型来源对上，Gemini Flash 表现出 +0.56 的自我偏好，Gemini Pro 却是 -0.22（偏好另一家族的输出），GPT-4o 更是以 -0.48 强烈偏好 Claude 的回答。[^2] 当前模型里的自我偏好，看起来不太像字面意义上的自我识别，更像取决于裁判最近是对着哪个训练分布优化的。别假定方向，去测。

### 修法：换家族裁判，或做混淆

默认规则很直接：别让裁判来自与被评系统同一个模型家族。评一个基于 Claude 的产品，就用 GPT 或 Gemini 的裁判。评一个基于 GPT 的产品，就用 Claude。成本差异很少是决定性因素，你打分的有效性才是。

在躲不开的时候（团队统一在一家供应商上、评测 rubric 是照着某个裁判写的、延迟预算只容得下一个模型），次优的缓解是作者混淆：在评判前把候选输出改写或重排一遍，抹掉风格指纹。这比听起来难，因为改写器自己也有风格偏好，但它确实显著削弱了裁判用来做自我识别的信号。[^5]

## 风格偏差

这一项没进 2023 年那批论文，而它现在是当前前沿裁判里最大的可测偏差。Soumik（2026）用 2024 年前后的五个模型测了受控配对：同样的内容，一版渲染成 markdown（标题、列表、加粗），一版是纯文本。每个模型都偏好 markdown 版，偏差分数在 0.76 到 0.92 之间（1.0 表示永远选 markdown）。[^2] 对比同一项研究里不到 0.04 的位置偏差。风格现在是主导性的失真，而几乎没人在实践文章里提它。

生产上的后果很直接：在 AlpacaEval、Chatbot Arena 这类 benchmark 上，输出风格偏 markdown 的模型可以单靠排版取胜。更糟的是，如果你拿一个有 markdown 偏好的裁判当奖励信号，你就是在训生成器往不该加标题和列表的地方加标题和列表。用户拿到一堆要点，而那个问题只想要一句话。

### 修法：先归一化，再评判

两个方案都管用，选哪个取决于你的生产系统有没有固定格式：

- **评判前归一化。** 把双方候选的 markdown 都剥掉（或者都渲染成 markdown），再送给裁判。这把偏差从变量里删掉了。
- **在提示词里要求。** 加一句：“不要根据排版偏好回答，只评内容质量。”这有帮助，但单靠它消不掉偏差。

Soumik 的 S8 组合策略（交换位置 + 思维链 + 明确 rubric）把五个受测模型的平均风格偏差从 0.84 压到 0.58，整体 MT-Bench 准确率提升 11.2 个百分点，p<0.0001。[^2] 在整项研究里，思维链提示（让裁判先推理再下判定）是唯一一个对每个模型、每个 benchmark 都不掉分的干预，而且不多花一次 API 调用。如果你只能做一件事，就做这个。

当你的裁判和候选系统在生产里永远输出同一种格式时，风格归一化可以跳过：markdown 对 markdown 没问题，纯文本对纯文本也没问题。这时偏差在两边是同一个常数偏移，而不是两者之间的失真。

## 标定：持续运转的闭环

上面每一个数字都来自论文。没有一个是来自你自己的裁判、你自己的数据——而只有那个数字才能决定你的评测流水线是不是在做正确的决策。标定这门功课，就是持续地、而不是一次性地测量这个差距。

在生产里站得住脚的节奏：

- **每周：** 拿一小批固定的、带已知人工标签的黄金集跑自动金丝雀。跟踪供应商侧模型静默更新引起的裁判漂移。
- **每月：** 从真实生产流量（不是金丝雀集）里抽 50 到 100 条分层样本做人工抽查。按裁判判定分层（判 A、判 B、判平局的样本各取等量），免得占比过低的类别消失。
- **每季度，或出现红线漂移时：** 用领域专家标注做一次完整重标定，并计算裁判与人类之间的 Cohen's kappa。[^6]

标定指标是 Cohen's kappa，不是原始一致率。原始一致率在不均衡的标签分布上看着挺好：如果你的真实输出有 90% 是“A 胜”，一个永远答“A”的裁判能拿到 90% 的原始一致率，而 kappa 接近零。实践者用的阈值是：rubric 稳定的情况下 kappa 低于 0.60 触发重标定；连续多个周期 kappa 持续低于 0.40，说明裁判或 rubric 需要重做。[^6]

```python
from dataclasses import dataclass
from typing import Literal

Verdict = Literal["A", "B", "tie"]

@dataclass
class Sample:
    judge_verdict: Verdict
    human_verdict: Verdict | None = None

def cohen_kappa(samples: list[Sample]) -> float:
    """Cohen's kappa between judge and human verdicts.
    Returns 0.0 when no humans have reviewed yet.
    """
    labels = ["A", "B", "tie"]
    reviewed = [s for s in samples if s.human_verdict is not None]
    n = len(reviewed)
    if n == 0:
        return 0.0
    po = sum(s.judge_verdict == s.human_verdict for s in reviewed) / n
    pe = sum(
        (sum(s.judge_verdict == lbl for s in reviewed) / n)
        * (sum(s.human_verdict == lbl for s in reviewed) / n)
        for lbl in labels
    )
    return (po - pe) / (1 - pe) if pe < 1 else 0.0
```

在供应商允许的范围内锁定裁判模型版本（OpenAI 的日期快照、Anthropic 的固定版本）。锁不了的时候，金丝雀跑是你唯一的信号，能告诉你评分分布已经在你脚下被静默更新挪走了。kappa 连续三个周期下滑才是要查的触发点；单独一周变差是噪声。

放到架构尺度上，[评测与可观测性流水线](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md)讲的是这些标定信号怎么变成生产指标、告警阈值和 trace 抽样。

## 参考文献

[^1]: Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena", NeurIPS 2023 Datasets and Benchmarks Track, arXiv:2306.05685, 2023 年 6 月, https://arxiv.org/abs/2306.05685
[^2]: Sadman Kabir Soumik, "Judging the Judges: A Systematic Evaluation of Bias Mitigation Strategies in LLM-as-a-Judge Pipelines", arXiv:2604.23178v1, 实验时间为 2025 年 4—6 月, https://arxiv.org/abs/2604.23178
[^3]: Keita Saito, Akifumi Wachi, Koki Wataoka, Youhei Akimoto, "Verbosity Bias in Preference Labeling by Large Language Models", arXiv:2310.10076, 2023 年 10 月, https://arxiv.org/abs/2310.10076
[^4]: Yann Dubois, Balazs Galambosi, Percy Liang, Tatsunori B. Hashimoto, "Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators", COLM 2024, arXiv:2404.04475, 2024 年 4 月, https://arxiv.org/abs/2404.04475
[^5]: Arjun Panickssery, Samuel R. Bowman, Shi Feng, "LLM Evaluators Recognize and Favor Their Own Generations", ICML 2024, arXiv:2404.13076, 2024 年 4 月, https://arxiv.org/abs/2404.13076
[^6]: Pratik Bhavsar, "How to Calibrate Your LLM Judge Using Human Annotations", Galileo Engineering Blog, 2026 年 5 月 15 日, https://galileo.ai/blog/calibrate-llm-judge-human-annotations
