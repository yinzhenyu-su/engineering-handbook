---
title: "如何选模型与读懂基准测试"
description: "公开榜单预测不了你的产品质量。按档位筛出候选、用自己的评测集实测、然后定下来。整个流程就这三步。"
slug: 05-choosing-models-and-reading-benchmarks
part: 2
chapter: 5
difficulty: beginner
prerequisites:
  - part-1-how-models-behave/00-the-model-landscape
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
tags: [llm, http-api]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 如何选模型与读懂基准测试

2026 年 2 月，OpenAI 公开下线了 SWE-bench Verified——业界被引用最多的编程基准之一。两项发现逼他们动手。第一，被审计的“失败”用例里有 59.4% 其实是测试在拒绝本来能跑的代码。第二，只给一行问题描述，GPT-5.2 就能复现好几个 Django 任务完整的 gold patch diff，那是背题，不是解题。[^1] 那个榜首的六个模型之间只差 1.3 个百分点。[^2] 团队把噪声当信号读，还据此上线。

这就是本章的一段话版：对任何一个具体的产品任务，你在选模型前查的那个榜单，预测力都不如你自己数据里的 50 个样例。下面说为什么，以及替代它的工作流。

## 榜单衡量的分布和你的产品不是同一个

从 LMArena 到 MMLU 再到 GPQA Diamond，每个公开榜单都在一个固定题库上给模型打分。Arena 的提示词偏向日常闲聊、随手写点代码、规划旅行路线。MMLU 是本科水平的选择题。GPQA Diamond 是专家级科学题。你的产品是合同条款、病历记录，或者某类特定的客服工单分流。这些分布几乎不重叠。

![共享一条横轴的两条重叠钟形曲线，横轴标注为提示词难度与领域专一程度；左边较宽较扁的曲线标注为 Arena 提示词分布，右边较窄较高的曲线标注为你的产品任务，两条曲线交汇处有一小块阴影重叠区](../../../ai-eng/part-2-working-with-model-apis/images/05-distribution-gap.png)
*榜单排名是在左边那条曲线上取平均；你的产品活在右边那条曲线上。只有那一小段重叠区，公开数字才搬得过来。*

Arena 自己的主题聚类分析就显示排名会随领域重排：在旅行行程上 GPT-4 和 Llama-2-70b-chat 几乎打平，但在编程与推理类聚类上，两者差距最大能拉到 43 个百分点。[^3] 全球排第二的模型在你的特定领域里压过排第一的那个，这种事发生的频率高到——从统计上说，在前 10% 这一档里按全球平均排名来选，已经接近掷硬币。

除了分布问题，Arena 还有个结构性问题。Singh 等人（《The Leaderboard Illusion》，2025）记录到：Meta 在发布前私底下把至少 27 个未公开的 Llama-4 变体放进 Chatbot Arena 测试，只留最好的那个分数。[^4] OpenAI 和 Google 各拿到全部 Arena 对局的约 20%；83 个开放权重模型加起来只有 29.7%。即便只是多打一点对局，也能在 Arena-Hard 上换来最高 112% 的相对分数提升，所以能选择性披露的闭源厂商在结构上占优。[^4] 文章发表后 Arena 调整了政策，但选择性披露仍然部分可行。

用 Arena 分数把明显不行的模型筛掉——也就是最差的那四分之一。别用它在前 10% 里排序。相邻名次的置信区间互相重叠，而选择性披露带来的优化压力，把榜单推向了榜单自身的动力学，而不是你的产品。

## 工作流：先筛候选，再实测，最后定夺

“查榜单”的替代品是三个阶段，阶段之间有明确的关卡。每个阶段只回答一个问题，答案没到手就别往下走。

**阶段 1：按档位特征筛候选，不看分数。** 读厂商文档。挑出 2 到 4 个满足硬约束的候选：上下文窗口装得下你的输入；需要 JSON 就要求支持结构化输出；价格在你预算范围内；限流跟你流量的形态合得来。[模型全景图](../part-1-how-models-behave/00-the-model-landscape.md) 里的档位划分到这里就够了。默认从中间档挑。如果任务难度确实吃不准，再加一个前沿档和一个小的做上下界。

**阶段 2：用自己的数据做一个 50 到 200 条的评测集。** 不是基准题。不是“看起来像”你任务的合成提示词。是你真实产品里的真实输入，每条配一个确定性检查或 LLM 裁判评分规则，判定通过或不通过。Hamel Husain 和 Shreya Shankar 把通用评测库为什么替代不了说得很直白：“Generic evaluation metrics are everywhere... These metrics measure abstract qualities that may not matter for your use case.”[^5] 你要预测的是模型在你这个分布上表现如何；只有你的分布能产出这个预测。

评测集本身就是一门手艺，[为什么没有评测就上不了线](../../../ai-eng/part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md) 和 [你的第一个评测集](../../../ai-eng/part-4-eval-fundamentals/02-your-first-eval-set.md) 里讲得很细。对这套工作流来说，最小可用版本是：50 条有代表性的样例，二元通过/不通过，纳入仓库版本控制，10 分钟内能跑完。

**阶段 3：跑候选、打分、定夺。** 把同一个提示词发给每个候选。记录通过率、p50 与 p95 延迟、每次通过摊到多少美元。按你的产品事先声明为主的那条维度，选赢的那个。把决定写下来，附上评测运行的时间戳和模型固定的版本号。

骨架就几十行 Python。摆出来是想说：这里没有魔法，工作流就是一个循环加一个打分函数。

```python
# 把同一个提示词跑遍各候选，对照你的标准答案打分。
# 把 API 调用和打分函数换成你真实的实现。

CANDIDATES = [
    {"provider": "openai",    "model": "gpt-5.3-instant"},
    {"provider": "anthropic", "model": "claude-fable-5"},
    {"provider": "openai",    "model": "gpt-5.5"},
]

def score(output: str, gold: str) -> bool:
    # 领域相关。可以是 schema 校验、正则、LLM 裁判。
    return gold.lower().strip() in output.lower()

def evaluate(eval_set, candidates):
    results = {}
    for c in candidates:
        passes = 0
        for example in eval_set:
            out = call_model(c, example["prompt"])  # 你的 API 封装
            if score(out, example["gold"]):
                passes += 1
        results[c["model"]] = passes / len(eval_set)
    return results

# eval_set 是 50-200 个字典：{"prompt": "...", "gold": "..."}
print(evaluate(eval_set, CANDIDATES))
```

在这之上还有一个有用的自检：当你的任务排名和公开榜单不一致时，该扔掉的是公开数字，不是你的。检查只需要一次除法：

```python
def check_inversion(public_a, public_b, your_a, your_b):
    public_winner = "A" if public_a > public_b else "B"
    your_winner   = "A" if your_a   > your_b   else "B"
    if public_winner != your_winner:
        return f"INVERSION: leaderboard says {public_winner}, your task says {your_winner}. Trust your task."
    return f"AGREEMENT: both pick {your_winner}. Still verify the margin clears confidence intervals."

# GPQA Diamond 前四名彼此相差在 1.3 个百分点以内（2026 年 5 月）：
# 下面的榜单排名是噪声；你在合同任务上那 5 个点的差距才是信号。
print(check_inversion(0.936, 0.942, 0.84, 0.79))
```

榜单和你的评测集打架时，榜单是在一个不属于你的分布上取平均，而且往往还被污染又抬高了一层。

## 数据污染，一段话讲清

公开基准活在公开互联网上。它们的题目和答案被贴到 GitHub、Stack Overflow、Hugging Face、博客教程和论坛帖子里，这些文本又被爬进下一轮预训练。模型于是在训练时见过测试集，背下其中一部分，分数被抬高，底层能力并没有变强。Microsoft Research 的 MMLU-CF 把 MMLU 重建成无污染版本，GPT-4o 从原版的 85% 以上掉到重建集的 73.4%，约 12 个点的差距可以归因于泄漏。[^6] 检测很难：简单的 n-gram 重叠检查用改写就能绕过，Evasive Augmentation Learning 技术在躲过 n-gram 过滤器的同时，展示出最高 15% 的基准提升。[^7] OpenAI 下线 SWE-bench 是最干净的公开证据：GPT-5.2 从短短的任务描述里逐字复现了 gold patch，这是标签级别的背题。[^1] 实际推论：当一个基准比模型的训练截止时间还老、最高分又已经不动了，就把这些数字当成顶到天花板、没有信息量。有抗污染的替代品就用替代品（MMLU-CF 替 MMLU，SWE-bench Pro 替 Verified），或者干脆跳过公开基准，信你自己的评测集。

## 什么时候该换模型

上线之后，问题就从“用哪个模型”变成“要不要换”。经过生产检验还站得住的规则是：只有当针对你自己 trace 的错误分析——而不是榜单头条——指出模型就是失败所在，并且你的评测集确认候选模型能改善那一类失败时，才换。Husain 的说法是：“Does error analysis suggest that your model is the problem?”[^5]

大多数生产故障是提示词故障或数据质量故障，只是穿了件模型故障的外衣。一个团队如果每次竞争对手发布新 SOTA 就换模型，一个季度就花在重写集成、继承新回归上，产品一点没变好。厂商支持的话就固定版本：写 `gpt-5.5-2026-04-12`，不写 `gpt-5.5`。每次版本升级都重跑评测集。把没固定版本的模型字符串当成隐式依赖，就像生产里的 `latest` 标签。

[升级阶梯](../part-0-start-here/03-the-escalation-ladder.md) 里的升级规则在这里反着用：在爬向更贵的模型之前，先把便宜的台阶走完。更好的提示词、更好的上下文、更好的检索、更好的工具。如果做完这些错误分析仍然指向模型，再换。放到架构尺度上，[HLD 第 9 部分讲了网关与路由拓扑](../../hld/part-9-ai-ml-system-design/) 用于整层模型的选型。

榜单不是没用，只是它的职责很小。它告诉你哪些模型明显已经出局。除此之外，唯一诚实的信号是你在自己数据上量出来的那个，而上面这套工作流就是产出它的办法。

## 参考文献

[^1]: OpenAI，《Why SWE-bench Verified no longer measures frontier coding capabilities》，openai.com，2026 年 2 月 23 日。https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/
[^2]: kilocode.substack.com，《Because Your Benchmark Score Doesn't Pay the Bill》，2026 年 5 月。https://open.substack.com/pub/kilocode/p/kilobench-because-your-benchmark
[^3]: Chiang, W.-L. 等，《Chatbot Arena: An Open Platform for Evaluating LLMs by Human Preference》，arXiv:2403.04132，2024。https://arxiv.org/abs/2403.04132
[^4]: Singh, S. 等（Cohere Labs、Princeton、MIT），《The Leaderboard Illusion》，arXiv:2504.20879，2025 年 4 月。https://arxiv.org/abs/2504.20879
[^5]: Husain, H. 与 Shankar, S.，《LLM Evals: Everything You Need to Know》，hamel.dev，2026 年 1 月 15 日。https://hamel.dev/blog/posts/evals-faq
[^6]: Zhao, Q. 等（Microsoft Research），《MMLU-CF: A Contamination-free Multi-task Language Understanding Benchmark》，arXiv:2412.15194，2024 年 12 月。https://arxiv.org/abs/2412.15194
[^7]: Xu, C. 等，《Benchmark Data Contamination of Large Language Models: A Survey》，arXiv:2406.04244，2024。https://arxiv.org/abs/2406.04244
