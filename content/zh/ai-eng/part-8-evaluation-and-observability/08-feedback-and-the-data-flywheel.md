---
title: "反馈与数据飞轮"
description: "生产流量如何复利成更好的评估集与更好的产品，而不是变成一轮接一轮、每个 sprint 都在修 bug 的跑步机。"
slug: 08-feedback-and-the-data-flywheel
part: 8
chapter: 8
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/01-look-at-your-data
  - 01-llm-as-judge
  - 05-online-evaluation-and-ab-testing
  - 06-tracing
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 反馈与数据飞轮

两个 AI 团队在同一天上线了同一个产品。六个月后，其中一个团队在人数不变的情况下把失败率从 12% 压到 3%。另一个团队每个 sprint 还在救火，质量在“略好一点”和“略差一点”之间来回摆，取决于哪个 prompt 改动正在灰度里。两边一样拼命。但只有一个团队在转飞轮。

这就是本章要说的事。生产流量是你的团队这辈子能拿到的最大数据集。如果每修一个 bug 同时也扩充了评估集、校准了评审模型（judge）、挡住了下一次回归，投入就会复利。如果每修一个 bug 就只是一个被修掉的 bug，你就在跑步机上。把前者变成后者的机制是数据飞轮（data flywheel）：日志变成标注样本，标注样本变成更好的评估，更好的评估变成更好的产品，更好的产品带来更多流量。Eugene Yan 和几位合著者在 2024 年中给出了经典的四句描述：“用人工评估判断模型表现、找出缺陷；拿标注数据微调模型或改写 prompt；重复。”[^1] 难点不在理解它。难点在于大多数团队卡在第一步，因为他们默认用户会去点那个大拇指。

## 为什么大拇指按钮扛不住这个活

当有人让你“收集用户反馈”，第一反应就是给每条回答加上点赞和点踩。这是一天能做完的活。看上去也是正确做法。问题出在回来的东西上。

有公开记录的生产环境 LLM 部署显示，显式反馈率只占全部交互的 0.5% 到 5%，而且不同交互类型之间的密度能差 10 倍到 30 倍。[^2] 日活 1,000、反馈率 1% 的情况下，你一天大概收到 10 到 30 条评分。而要有统计置信度地检测出 5% 的质量变化，需要接近 1,000 个样本。所以你那条“反馈信号”，其实是延迟 30 天、有损的、对用户真实感受的一个远距离投影。

更糟的还在后面。点下去的那 1% 不是随机样本。他们落在满意度分布的两端：生气的用户猛戳点踩，铁粉猛戳点赞，中间那一大群一声不响。Morandi 和 Viswanathan 在 2026 年的一篇论文里把这个偏差的幅度算了出来：对显式反馈做朴素平均，结果可能偏离真实系统质量 40 到 50 个百分点。他们引用了一个有记录的生产部署：朴素的大拇指评分读出 34%，而还原出来的真实质量是 83%，差了 49 个百分点。[^3] 你的看板说你在失败。你并没有。你只是在误读一个自选择样本。

## 收集用户不用主动选择就会给出的信号

不管你做不做按钮，每个用户都在给你信号。他们会重试。会重新生成。会复制文本。会在发出去之前先改你的输出。会扔掉这个会话，三分钟后又开一个新会话，把同一个问题换个说法再问一遍。编程助手和 Midjourney 依赖的“接受／修改／忽略”（accept-or-modify-or-ignore）模式，以零 UI 成本覆盖了 100% 的用户。[^1] 尤其是重新生成按钮，它是你能拿到的最响亮的隐性点踩：用户看了你的回答，觉得连改都不值得改。

线上信号的拆解（哪些重试要计数、每种信号覆盖多大比例、如何去抖）是[线上评估那一章](./05-online-evaluation-and-ab-testing.md)的主题。这里要说的只有一点：这些信号要和 trace 记在一起，后面的标注流水线才能回查。这份 schema 不好看，但值得在第一天就定对：

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from typing import Optional

class ImplicitSignal(str, Enum):
    COPY = "copy"
    RETRY = "retry"
    EDIT = "edit"
    ABANDON = "abandon"
    FOLLOW_UP = "follow_up"

@dataclass
class FeedbackRecord:
    trace_id: str
    session_id: str
    timestamp: str
    explicit_rating: Optional[int]      # 1（通过）、0（失败），没有则为 None
    explicit_comment: Optional[str]     # 自由文本，只在提供入口时才有
    implicit_signal: Optional[ImplicitSignal]
    latency_ms: int
    model_version: str
    prompt_hash: str                    # prompt 模板的 sha256[:8]
```

日后能回本的是 `model_version` 和 `prompt_hash` 这两个字段。没有它们，你就分不清一次退化是来自模型更新、prompt 改动，还是用户行为漂移，飞轮就变成了考古。

显式渠道只做二值评分，外加一个可选评论。别搞五星李克特量表。Shankar 和 Husain 在这一点上说法一致：“‘有点帮助’和‘有帮助’到底差在哪？”[^4] 二值对用户来说更容易给得一致，对下游评审模型来说也更容易复现。可选自由文本里的批评意见，是闭环下一阶段的燃料；把二值判断和详细批评配在一起，相比只给判断，人与 LLM 评审的一致率能提高 15 到 20 个百分点——这是 Husain 截至 2025 年 3 月的客户实施情况。[^5]

## 会复利的那个闭环

一条记了反馈信号的 trace 只是原材料，不是改进。五个步骤把原材料变成更好的产品，而飞轮就是你让这五步按节奏跑起来之后得到的东西：

1. **全记下来。** 完整 trace：输入、输出、中间工具调用、延迟、模型版本、prompt hash。缺了这个，后面每一步都做不了。这套管道在[追踪那一章](./06-tracing.md)。
2. **采样与分诊。** 你永远不可能逐条看完所有 trace。按优先级挑：显式评分低的、重试之后放弃的会话、自动评审模型多次运行自相矛盾的 trace，再加一小份均匀随机样本保证无偏覆盖。
3. **对样本做错误分析。** 开放式编码写成自由文本笔记，主轴编码归并成一个小型失败分类法，再统计每个类别的数量。具体做法在[看看你的数据](../part-4-eval-fundamentals/01-look-at-your-data.md)，这里不重复。
4. **把发现变成评估资产。** 这是团队最常跳过的一步，也是飞轮和跑步机的分界线。对每个出现三次及以上的类别，写一条回归测试。对最烂的那几个样例，把修正后的输出作为 few-shot 示例加进对应评审模型的 prompt。
5. **拿扩充后的评估集来发布改动。** 新 prompt 或新模型只有在变大的测试集上没有退化才放出去。变好的系统产生更好的 trace，回流到第一步。

![左侧是一个标着五个飞轮阶段的顺时针闭环，右侧是一条水平的链路，反复出现“修 bug—修 bug”，没有任何路径回到评估](../../../ai-eng/part-8-evaluation-and-observability/images/08-flywheel-vs-treadmill.png)
*飞轮把每次修复变成一个可复用的测试；跑步机则永远在修同一种形状的 bug。*

两个生产案例让“复利”变得具体。一篇 EMNLP 2025 Industry Track 的论文来自美国某企业级客户支持部署，他们把四种标注类型直接嵌进坐席的工作流，让客服在干活的过程中顺手把 trace 标了。这个飞轮把检索准确率提升了 11.7%（recall@75）和 14.8%（precision@8），生成有用性提升 8.4%，采纳率提升 4.5%，同时把重训练周期从几个月缩短到几周。[^6] DoorDash 2026 年 6 月的仿真与评估平台换了个角度：把生产客服对话转成可回放的测试场景，用一个扮演客户的 LLM 驱动多轮对话去打这个 chatbot，任何 prompt 改动在碰到真实流量之前先过一遍。他们的招牌数字是：302 场模拟对话 5 分钟跑完，而 175 场生产对话要摊在 7 小时里；模拟出来的转人工率 46%，和生产观测到的 44% 对得上；chatbot 开发期间的幻觉减少 90%。[^7]

两个团队共有的形状都在第 4 步：一个修掉的 bug 不会只是一个修掉的 bug。它变成一条此后每次改动都要跑的测试。这就是全部的复利机制。

## 当地基在你脚下移动时重新标注

评估集越来越大，听起来纯是好消息，直到你发现二月写下的标注已经不太符合六月你会用的标准。有三样东西一直在动：

- **协变量漂移（covariate shift）。** 新用户、新话题、新产品功能。输入变了，什么算好答案没变。
- **概念漂移（concept drift）。** “好”本身的含义变了。一个 2024 年很出色的金融助手，到 2026 年是错的，因为利率变了。
- **标准漂移（criteria drift）。** 在看到几百条输出之前，你自己也不知道自己要什么。Shankar 和同事在 2024 年把这件事命名了：“用户需要有标准才能给输出打分，但给输出打分又帮用户把标准定义了出来。”[^8]

只有后两种需要重新标注旧样本。协变量漂移只是要从输入空间的新区域拿新样本。在生产里把这两者分开，是重标注调度器的第一件工作。

最便宜又值得接上的信号，是近期输入 embedding 分布与参考窗口之间的 Jensen-Shannon 散度。Shankar 推荐的触发阈值是 0.05。[^4] 再叠一个日历下限（不管怎样每周都看一些 trace），你就有一条站得住的调度规则：

```python
import math
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Optional

@dataclass
class DriftEvent:
    js_divergence: float
    triggered: bool
    reason: str

def js_divergence(p: list[float], q: list[float]) -> float:
    m = [(pi + qi) / 2 for pi, qi in zip(p, q)]
    def kl(a, b):
        return sum(ai * math.log(ai / bi) for ai, bi in zip(a, b) if ai > 0 and bi > 0)
    return 0.5 * kl(p, m) + 0.5 * kl(q, m)

def check_drift(
    reference: list[float],
    current: list[float],
    last_review: Optional[datetime],
    threshold: float = 0.05,
    cadence_days: int = 7,
) -> DriftEvent:
    jsd = js_divergence(reference, current)
    now = datetime.now(timezone.utc)
    cadence_due = last_review is None or (now - last_review).days >= cadence_days
    reasons = []
    if jsd >= threshold:
        reasons.append(f"JS divergence {jsd:.3f} >= {threshold}")
    if cadence_due:
        reasons.append(f"{cadence_days}-day cadence elapsed")
    return DriftEvent(jsd, bool(reasons), "; ".join(reasons) or "no trigger")
```

每天在近期输入 embedding 的滚动窗口上跑一次散度检查。一旦触发，那一周的评审就向漂移区域不均衡地多采样。Voiceflow 的经历是反面教材：一次从 `gpt-3.5-turbo-0301` 到 `gpt-3.5-turbo-1106` 的例行迁移，让他们现有评估集上的意图分类准确率掉了大约 10%。[^1] 只绑日历的调度器会漏掉这次。绑模型版本变化的调度器，会在新模型上线当天就抓住它。

还有两个习惯能防止评估集发霉。把标注样本当成有版本的产物：带上时间戳、标注时所依据的评分细则（rubric）版本，以及标注人。评分细则变严时，你不必把旧标注扔掉；按新细则重新校验一份随机样本，再把不一致的地方补标回来。任何从标注池里取样的 few-shot 检索都要给新鲜度加权；上周修正过的一条 trace，比一年前那条更能示范“好长什么样”，哪怕余弦相似度不这么认为。[^4]

这件事之所以格外要紧，是因为评估集悄悄过期是看不见的。覆盖率看起来一样。评审模型照样打分。看板照样是绿的。而与此同时，标注当初依据的那份细则，已经不是今天这屋里任何人会写的细则了，每一次 prompt 改动都在被一个幽灵评分。定期重标注，才能让飞轮一直拉着真实的东西转。

## 参考文献

[^1]: Eugene Yan, Bryan Bischof, Charles Frye, Hamel Husain, Jason Liu, and Shreya Shankar, "What We've Learned From A Year of Building with LLMs," applied-llms.org, June 8, 2024. https://applied-llms.org
[^2]: Andrea Morandi and Mahesh Viswanathan, "Correcting Selection Bias in Sparse User Feedback for Large Language Model Quality Estimation," arXiv:2605.12177, May 2026. https://arxiv.org/html/2605.12177v1
[^3]: Morandi and Viswanathan, ibid., Section V.A and Table 1.
[^4]: Shreya Shankar, "Data Flywheels for LLM Applications," sh-reya.com, July 1, 2024. https://sh-reya.com/blog/ai-engineering-flywheel/
[^5]: Hamel Husain, "A Field Guide to Rapidly Improving AI Products," hamel.dev, March 24, 2025. https://hamel.dev/blog/posts/field-guide/
[^6]: Cen Mia Zhao et al., "Agent-in-the-Loop: A Data Flywheel for Continuous Improvement in LLM-based Customer Support," arXiv:2510.06674, EMNLP 2025 Industry Track, October 2025. https://arxiv.org/abs/2510.06674
[^7]: DoorDash Engineering, "Inside DoorDash's one-click simulation and evaluation platform for support chatbots," careersatdoordash.com, June 1, 2026. https://careersatdoordash.com/blog/doordashs-one-click-simulation-and-evaluation-platform-for-support-chatbots/
[^8]: Shreya Shankar, J.D. Zamfirescu-Pereira, Bjorn Hartmann, Aditya G. Parameswaran, and Ian Arawjo, "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences," arXiv:2404.12272, ACM UIST 2024. https://arxiv.org/abs/2404.12272
