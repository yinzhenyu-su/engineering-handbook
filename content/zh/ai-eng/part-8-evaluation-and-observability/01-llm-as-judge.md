---
title: "LLM-as-judge"
description: "构建一个你能信任的第二 LLM 评判者：用二值的通过/不通过取代李克特量表、提示词该怎么写，以及上线前必须跑通的人工一致性回路。"
slug: 01-llm-as-judge
part: 8
chapter: 1
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
  - part-4-eval-fundamentals/02-your-first-eval-set
  - part-4-eval-fundamentals/03-assertions-and-unit-tests
  - part-3-prompt-engineering/00-anatomy-of-a-production-prompt
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# LLM-as-judge

有个团队在客服助手上线三个月后，看板上挂着八项评分：有用性 3.7，语气 4.1，准确性 3.9，清晰度 4.0，完整性 3.5，安全性 4.8，说服力 3.2，共情 3.8。这些分数周与周之间几乎不动。没人说得清 3 和 4 差在哪。PM 问：“这次提示词改动让产品变好了吗？”没人答得上来。这块看板漂亮、昂贵，本质上是一场表演。

这块看板就是李克特量表（Likert scale）式的 LLM-as-judge 默认会产出的东西，也是大多数团队在一个季度之内就放弃评判者的原因。解法不是换个模型，而是换个问法。把八个 1 到 5 分换成一个二值判定，而且这个判定只有在你测出它与一位你信任的人类标注者的一致率之后，才允许上线。这就是本章要讲的事。

LLM-as-judge 就是在主流程之外多挂一次模型调用。你把输入和输出交给它，描述清楚什么叫“好”，它返回一个判定。它的吸引力在于规模：校准好之后，一个评判者一天能给上千条轨迹（trace）打分，而这点时间人类评审只够读三条。风险在于，评判者是一个模型在评判另一个模型。它会出错、有偏见，还会以你除非主动去查否则永远看不到的方式稳定地自相矛盾。[LLM 输出的断言与单元测试](../part-4-eval-fundamentals/03-assertions-and-unit-tests.md)里的确定性检查能抓到它们能抓的部分。评判者存在的意义是剩下的那些：语气、对来源的忠实度、这次到底有没有帮上忙，以及那些真实存在却塞不进正则的失效模式。

## 二值判定比李克特打分更好

第一个决策是判定用什么形态。在生产里翻车的形态是 1 到 5（或 1 到 10）的李克特打分，它翻车有三个具体原因，跑过这套东西的团队都在文档里记下了。

- **标注者分不清 3 和 4。** 让你自己的两个工程师标注同一条回复，一个说 3，一个说 4，两人都讲不出差别在哪。评判者继承的是同一团模糊。Hamel Husain 在 2024 年跟 30 多家公司一起设计评判者的评测，发现这种分歧是普遍存在的，不把刻度压缩掉就修不好。[^1]
- **这个数字没法指向行动。** 一个 3 不会告诉你该改什么。一个带书面评语的“不通过”会精确告诉你该改什么。
- **八个维度八个分数，就是指标泛滥。** 开头那个团队有八个没人信的分数。一个经过充分验证的二值数字，比八个模糊数字更值钱。

替代方案是一个二值问题，围绕真正的产品结果来提问。不是“请给有用性打 1 到 5 分”，而是“助手有没有解决用户的问题，是或否？”评判者返回 `pass` 或 `fail`，外加一段理由。这样与人类相比的 Cohen's kappa 就可计算了；通过率就可跟踪了；回归就看得见了。

有两种情况可以走例外通道。第一，当你确实需要在 N 个候选之间排名（RLHF 数据收集，或者“更好”比“可接受”更重要的 A/B 测试），成对比较（pairwise comparison，“A 比 B 好吗？”）才是对的基本单元，它同样是二值的。第二，当序数排名本身就是产品（重排模型、推荐系统），带 token 概率的 G-Eval 式打分可以活下来，但你仍然要用同样的方式对照人工标签做验证。[^2] 这两种之外，默认用二值通过/不通过。[^3]

## 判题提示词由哪几部分组成

一条能用的判题提示词有五部分，少任何一部分，你都要在后面的迭代里把丢掉的一致率一点点抠回来。

1. **角色声明。**“You are an expert evaluator.”朴素就行，不要堆砌辞藻。角色在模型看到评分标准之前，先把它的身份定位好。
2. **一条表述精确的标准。**不是八条。只写领域专家最在意的那一个属性：“这条 SQL 查询有没有正确回答用户的自然语言问题？”如果你挑不出一个，说明你还没搞清自己在评什么。
3. **带评语的少样本示例。**四到八个示例，通过和不通过都要覆盖，每个都附书面理由。评语是承重的部分。它不是装饰，它是评判者学会泛化的方式。省评语是评判者卡在 70% 一致率最常见的原因。
4. **一条思维链指令。**“先逐步推理，再给出判定。”Zheng 等人在 NeurIPS 2023 的研究发现，加上带参考答案的思维链提示后，GPT-4 在数学批改上的失败率从 70%（20 例里 14 例）降到 15%（20 例里 3 例）。[^4] 先推理再判定是地板，只给判定是天花板。
5. **结构化输出格式。**JSON，两个键：`rationale`（字符串）和 `verdict`（`"pass"` 或 `"fail"`）。把它解析出来，两个都存下来，让看板去聚合判定，工程师只在看到异常时去读理由。

下面是拼装函数，锋利程度足够直接抄进项目：

```python
import json

def build_judge_prompt(
    criteria: str,
    examples: list[dict],
    input_text: str,
    output_text: str,
) -> list[dict]:
    """Binary pass/fail judge with CoT rationale and few-shot critiques."""
    system = (
        "You are an expert evaluator. For each interaction, reason step by "
        "step, then output a JSON object with keys 'rationale' (string) and "
        f"'verdict' (either 'pass' or 'fail'). Criteria: {criteria}"
    )
    messages = [{"role": "system", "content": system}]
    for ex in examples:
        messages.append({
            "role": "user",
            "content": f"Input: {ex['input']}\nOutput: {ex['output']}",
        })
        messages.append({
            "role": "assistant",
            "content": json.dumps({
                "rationale": ex["rationale"],
                "verdict": ex["verdict"],
            }),
        })
    messages.append({
        "role": "user",
        "content": f"Input: {input_text}\nOutput: {output_text}",
    })
    return messages
```

关于模型有两点实务提醒。在你付得起的范围里用最强的评判者，哪怕它比你的应用模型还强；Husain 的原则是“在我的成本/延迟预算内用最强的模型”，而这份预算跟你的主模型是分开算的。[^1] 参数量低于 7B 的开源评判者没法稳定产出可解析的 JSON，这让它们在 CI 里根本没法用。另外，当评判模型和应用模型属于同一家族时，不必纠结自我偏好偏差（self-preference bias）：它确实存在（在 LMSYS 的研究里 GPT-4 相对人类评审给自己高估了 10%），但幅度小到只要人工一致率高就能盖过去。[^4] 真正会咬人的偏差是位置偏差（position bias）和冗长偏差（verbosity bias），它们在[评判者的失效模式](./02-judge-failure-modes.md)里单独讲；简版做法是成对调用跑两遍并互换位置，再用大白话明确告诉评判者不要偏爱更长的回答。

## 本章真正的内容是验证回路

没跟人类验证过的评判者不叫评判者，它是个带着自信问题的观点生成器。把前者变成后者的那个回路，小到能画在一块白板上，严谨到能发表：采样、人工标注、让评判者跑同一批样本、测一致率、改提示词，如此重复直到可以上线。

![一个五节点的环形回路，展示用人类专家验证 LLM 评判者的迭代过程：采集 30 到 200 条交互样本，由专家标注通过/不通过并写评语，让 LLM 评判者跑同一批样本，计算 Cohen's kappa 与百分比一致率，然后要么改进判题提示词回到回路起点，要么在一致率越过阈值时进入上线。](../../../ai-eng/part-8-evaluation-and-observability/images/01-validation-loop.png)
*验证回路。你要一直待在循环里，直到一致率高到匹配风险；对 Honeycomb 的 Query Assistant 来说，这花了三轮迭代才把一致率推到 90% 以上。这个回路永远不会真正结束，换模型、改提示词、用户行为漂移，都会把你送回第一步。*

**第 1 步：收集 30 到 200 条轨迹。**有真实生产流量就从生产里取；没有就让合成的提示词跑一遍你的应用。要做分层，让样本看起来像你实际服务的分布，而不是一份精选的高光合集。Husain 给出的可操作下限是 30 条；当你标注时不再看到新的失效模式，就说明采够了。[^1]

**第 2 步：让首席领域专家标注每一条。**通过或不通过，外加一到三句说明理由的评语。不是初级评审，不是众包。是那个品味定义了产品的人。这些评语在干两件事：它们成为评判者的标准答案，同时还是你未来的少样本示例。如果专家写评语时很吃力，说明标准本身还不清晰——这个发现越早越好，别等到上线三周后才发现。

**第 3 步：让评判者跑同一批样本。**同样的输入，同样的输出，同样的提示词。判定和理由都存下来。

**第 4 步：测一致率。**两个数字，不是一个。

```python
from collections import Counter

def percent_agreement(human: list[str], judge: list[str]) -> float:
    """Fraction of items where human and judge agree."""
    assert len(human) == len(judge)
    return sum(h == j for h, j in zip(human, judge)) / len(human)

def cohen_kappa(human: list[str], judge: list[str]) -> float:
    """Agreement adjusted for chance. kappa = (p_o - p_e) / (1 - p_e)."""
    assert len(human) == len(judge)
    n = len(human)
    classes = list(set(human) | set(judge))
    p_o = percent_agreement(human, judge)
    h_counts = Counter(human)
    j_counts = Counter(judge)
    p_e = sum((h_counts[c] / n) * (j_counts[c] / n) for c in classes)
    return 1.0 if p_e == 1.0 else (p_o - p_e) / (1 - p_e)
```

百分比一致率是拿去给利益相关方汇报的。Cohen's kappa 才告诉你真相。kappa 校正了巧合：如果你的输出有 90% 都能通过，一个永远说“通过”的评判者也能拿到 90% 的原始一致率，而 kappa 接近零。Eugene Yan 2024 年对评判者相关论文的综述里举了一个真实例子：一个 llama-3-8b 评判者与人类的一致率达到 80%，但 kappa 只有 0.62，意思是它在真正重要的失败案例上几乎跟瞎猜差不多。[^2] kappa 用 Landis-Koch 约定解读：0.41 到 0.60 是中等，0.61 到 0.80 是显著，0.80 以上是几乎完全一致。

**第 5 步：读那些分歧。**人类说通过而评判者说不通过（或反过来）的地方，去看理由。是评判者缺了标准里没写明的上下文？还是人类在用一条标准里没捕获的准则？两种都会发生，而且常常发生在同一轮里。Shankar 等人在 UIST 2024 把这件事命名为*标准漂移*（criteria drift）：给样本打分这个动作本身，就是专家在把自己心里的“好”磨清楚，所以你第一轮写下的标准到第二轮就是错的，这不是 bug。[^5] 更新标准。更新少样本示例。再跑一遍。

## 评判者什么时候可以上线？

没有通用阈值，谁给你报一个具体数字，都是在卖东西。诚实的版本是一个下限加一个上限，都钉在你系统的风险上。

下限：如果一个评判者与你专家之间的 Cohen's kappa 低于 0.6，别上线。这是 Landis-Koch 里中等与显著一致率的分界，低于它，评判者太不可靠，不能当主信号用。多数生产场景的目标是 kappa 0.6 到 0.8 左右，或者在类别平衡的二值任务上大约 80% 到 90% 的原始一致率。

上限：对能拿到什么要现实。Thakur 等人在 2024 年用 TriviaQA 对照人类评估 GPT-4，测得 GPT-4 的 kappa 是 0.84，LLaMA-3-70B 是 0.79，而同一任务上人类之间的一致率 kappa 是 0.97。[^6] 即便在一个正确答案毫无歧义、还有参考答案的任务上，最好的 API 级别评判者也比人类之间的一致率差一点点。在主观任务上（有用性、语气、连贯性），差距更大。等 kappa 超过 0.95，你会永远上不了线。

一份可用的评分基准：

| 用途 | 目标 kappa | 说明 |
|---|---|---|
| 内部看板、批量评测 | > 0.6 | 够用来跟踪提示词版本之间的相对变化。 |
| 面向客户应用的生产监控 | > 0.7 | 争取约 90% 的百分比一致率；Honeycomb 的 Query Assistant 用三轮迭代达到了。[^1] |
| 高风险领域（医疗、法律、金融） | > 0.8 | 并且要永久保留人工复核在回路里。评判者是增强，永远不是替代。 |

越过你的阈值，评判者就可以上线，但回路永远不会真正闭合。新的模型快照、应用里的提示词改动、用户行为的变化：这三样都会让校准失效。每个季度、每次换模型、以及每次评判者的通过率出现应用本身没挣来的突变时，都要在新样本上重新测一遍。

## 什么时候干脆别用评判者

当更便宜的东西能干同样的事，评判者就是错的工具。如果这个属性你能用正则、JSON schema、SQL 解析或单元测试表达出来，那就用它。代码断言是确定性的、免费的、毫秒级返回；评判者是有概率的、每次调用都花真钱、秒级返回。[LLM 输出的断言与单元测试](../part-4-eval-fundamentals/03-assertions-and-unit-tests.md)里的金字塔能抓住一切你写得下来的东西。评判者是给剩下的部分用的：语气、忠实度、用户到底有没有拿到他要的东西。那些重要却塞不进正则的失效模式。

评判者同样不适合放进在线路径。评判者会给每个请求加一整趟 LLM 往返的延迟，这在异步评测流水线上没问题，在面向用户的路径上不可接受。把评判者放在带外跑，跑在生产轨迹的采样上，结果流进看板和告警。applied-llms.org 那批人（Husain、Yan、Liu、Shankar 等人，2024 年 6 月总结各家一年多的生产经验）说得很直接：LLM-as-judge 不是银弹，在线路径上尤其弱，需要精确数值核验的任务上弱，没有领域专家可用来验证时也弱。[^3] 框架自带的现成指标（DeepEval 的通用“answer relevancy”、Ragas 的“faithfulness”）作为起点基线是有用的，但它们评的是通用属性，不是你的产品的成功标准。它们替代不了验证回路。它们最多只能当起步提示词，然后你拿自己的人工标签去验证它、再迭代。

能在你的技术栈里站住脚的评判者，是你自己搭起来、验证过、并且会持续复验的那个。不是框架开箱给你的那个。

放到架构尺度上，[评测与可观测性流水线](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md)讲了评判者怎么接进轨迹存储、异步打分 worker，以及领域专家天天待着的标注队列。

## 参考文献

[^1]: Hamel Husain, "Creating an LLM-as-a-Judge That Drives Business Results," hamelhusain.substack.com, October 29, 2024, https://hamelhusain.substack.com/p/llm-judge
[^2]: Eugene Yan, "Evaluating the Effectiveness of LLM-Evaluators (aka LLM-as-Judge)," eugeneyan.com, August 2024, https://eugeneyan.com/writing/llm-evaluators/
[^3]: Bryan Bischof, Hamel Husain, Charles Frye, Jason Liu, Shreya Shankar, et al., "What We've Learned From a Year of Building with LLMs," applied-llms.org, June 2024, https://applied-llms.org/
[^4]: Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena," NeurIPS 2023 Datasets and Benchmarks Track, https://arxiv.org/abs/2306.05685
[^5]: Shreya Shankar, J. D. Zamfirescu-Pereira, Bjorn Hartmann, Aditya G. Parameswaran, Ian Arawjo, "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences," UIST 2024, https://arxiv.org/abs/2404.12272
[^6]: Aman Singh Thakur, Kartik Choudhary, Venkat Srinik Ramayapally, Sankaran Vaidyanathan, Dieuwke Hupkes, "Judging the Judges: Evaluating Alignment and Vulnerabilities in LLMs-as-Judges," arXiv:2406.12624, June 2024, https://arxiv.org/abs/2406.12624
