---
title: "AI 产品指标"
description: "超越准确率。任务成功率、分流与承接与真正解决的区别、把升级当作质量信号、完成时长，以及 AI 功能留存。"
slug: 02-ai-product-metrics
part: 11
chapter: 2
difficulty: intermediate
prerequisites:
  - part-8-evaluation-and-observability/06-tracing
  - part-8-evaluation-and-observability/07-monitoring-and-dashboards
  - 01-failure-recovery-ux
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# AI 产品指标

一个团队上线了 AI 功能，并像给其他所有功能做埋点那样给它做埋点：延迟直方图、token 计数、错误率，全部接到 OpenTelemetry。三个月后，仪表盘一片绿，留存却在往下掉。OTel 的 span 把这通 API 调用的一切都告诉了他们，却完全没告诉他们用户的问题有没有被解决。

这就是本章要讲的缺口。模型准确率（accuracy）衡量的是一个 LLM 在孤立环境下能否产出技术上正确的输出；产品指标衡量的是用户有没有办成他本来要做的事。两者在方向和量级上都会背离：一个在基准测试上拿 90 分的系统，任务成功率可能只有 40%——如果用户没法把自己的目标表述成基准题的形式，如果系统没法执行完成任务所需的动作，或者每一步都慢到用户放弃。

这里有五个指标承担主要分量，它们和你的 span 不在同一层：任务成功率（Task Success Rate）、分流率（Deflection Rate，以及大家混着用的三种定义）、升级率（Escalation Rate）、完成时长（Time-to-Completion）和功能留存（Feature Retention）。每一个都需要 LLM 调用本身不会免费给你的埋点。

## 两个层次，一个关联键

OpenTelemetry 的 GenAI 语义约定（v1.41.1，截至 2026 年 6 月状态为 Development）为 LLM 调用标准化了 span 属性：`gen_ai.client.operation.duration`、`gen_ai.response.time_to_first_chunk`、`gen_ai.usage.input_tokens`、`gen_ai.usage.output_tokens`、`gen_ai.conversation.id`。[^1] 这些描述的是 API 调用，不描述用户拿它做了什么。

产品分析层才是记录真实发生了什么的地方，session 事件就在那里（Amplitude、Mixpanel、你自己的管道）。用户接受了建议吗？重新生成了吗？完成了下游动作吗？明天还回来吗？两层通过共享的 `request_id` 关联，或者通过两个系统都透传的 `gen_ai.conversation.id` 关联。把两层关联起来，是本章每一个指标的前提。

![两条平行的纵向事件流，分别标注为“AI infrastructure”和“Product analytics”，中间由一个带标签的关联箭头横向连接，显示 request_id 如何把一条 LLM span 与该用户的下游动作和结果事件绑定](../../../ai-eng/part-11-ai-product/images/02-two-layer-instrumentation.png)
*OTel span 在左边，讲的是这次调用；产品事件在右边，讲的是结果。关联键是让任何一边变得有用的唯一东西。*

搞错这一点的团队会做出极其出色的基础设施仪表盘，然后管它叫产品仪表盘。它不是。延迟 SLO、成本追踪、错误率用 OTel；有没有把事情办成，用产品分析。

## 任务成功率，需要一个真正的判定谓词

英伟达（NVIDIA）的 agent 评估指南把 TSR 表述为“意图加约束”：“Update this record through this API within two tool calls. Measure success only when the agent fully resolves the intent within those constraints.”[^2] 没有明确的二元成功谓词，TSR 就没法算。没有谓词的团队会悄悄换成代理指标：点赞率、未升级率、LLM 裁判给出的回答质量分。每一个代理指标都是绿的，而真实指标在跌。

成功谓词必须是用户做出的下游动作，而不是 AI 说出的某句话。按功能类型，可用的默认值：

- **客户支持：** 用户 48 小时内没有再次联系。
- **代码助手：** 建议被原样提交，未做修改。
- **文档起草：** 文档被发布或分享。
- **搜索/RAG：** 用户点击了某个结果，*并且* 5 分钟内没有换个说法再问一次。

TSR 要求的另一项纪律是分群。一个不分群的单一数字，把基线难度差别很大的不同任务类型平均到一起，而这个均值恰好掩盖了哪一类在失败。按场景（正常、工具降级、指令含糊）分别跟踪 TSR，脆弱性才有地方暴露出来。

## 分流、承接与解决不是一回事

三个词常被混着用，但它们不是同一个数：[^3]

- **原始分流率（raw deflection rate）：** 未转人工就关闭的会话，其中包含放弃的用户。会被放弃行为抬高。
- **承接率（containment rate）：** 用户走到 AI 交互终点而没有升级的会话。是分流的严格子集，仍然不能证明问题被解决了。
- **真正解决率（true resolution rate）：** 承接，加上用户认可的结果（有下游动作、交互后 CSAT 达标、48 小时内没有再次联系）。

Voiceflow 的企业数据（2026 年 3 月）给出的现实区间是：账号管理这类定义清晰的 Tier 1 任务，头几个月承接率 70-90%；Tier 1 与 Tier 2 混合范围到六个月时 50-70%；包含复杂边缘情况的完整支持范围 40-60%。[^3] 厂商宣传的“80-90% 分流率”几乎都是在窄负载上用原始分流率算出来的。他们没说谎，只是用了最宽松的那个定义。

失效模式是“分流表演”。一个背着分流目标的团队，会把升级入口做得更难找，降低 AI 说“我不知道”的意愿，给转人工加摩擦。原始分流率上升，CSAT 下降，再次联系率上升。仪表盘还是绿的，因为这个指标从一开始就选错了。

能抓住这一点的纪律是：绝不单独报分流率，永远配上 24-48 小时内的再次联系率。如果 AI 处理过的会话再次联系率超过 15%，说明原始分流率被抬高了，你正在悄悄辜负那些被记成胜利的用户。经济账让这种诱惑变得很真实。微软 Copilot Studio（2026 年 1 月）给出的数字是：一次人工支持联系 5-10 美元，一次 AI 会话约 0.50 美元。[^4] 10 倍的成本比，足以腐蚀摆在它面前的几乎任何指标。

## 把升级率当作 AI 质量信号

多数组织把升级率读成人力规划指标。AI 团队应该把它读成质量信号，而多数团队并没有。[^5] 当 AI 团队上线了一次能力回退，四周内升级率涨了六个点，运营看到的是一道人头问题，于是再招两个客服。评估套件还是绿的，因为测试集没有采样到引发回退的那些新查询模式。根因永远没人查。

把升级拆成四类，信号才看得懂：

- **策略型升级：** AI 本可以处理，但路由规则要求转人工（超额退款、身份核验）。这类在各版本间应该保持平稳；出现趋势就是路由 bug。
- **能力型升级：** AI 试过了，调了工具，因为置信度低而选择交出去。*这才是 AI 团队的信号。*
- **拒答型升级：** AI 出于安全考虑拒绝。需要内容复审，因为过度拒答在队列计数里是看不见的。
- **放弃型升级：** 用户中途放弃，从另一个渠道找到了你。需要跨渠道的 session 关联才能发现。

阈值告警这种模式发得太晚。能提前抓住漂移的，是 SRE 的多窗口燃烧率告警：短窗口（6 小时）和长窗口（7 天）同时高于规划基线的某个倍数。六周里从 18% 漂到 24%，应该在第三周就响，而不是到第八周出现在一份人力备忘录里。

```python
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class EscalationDataPoint:
    timestamp: datetime
    escalated: bool
    cohort: str

def burn_rate_alert(points, short_h=6.0, long_h=168.0,
                    baseline=0.22, multiplier=1.3, now=None):
    now = now or datetime.utcnow()
    short_pts = [p for p in points if p.timestamp >= now - timedelta(hours=short_h)]
    long_pts = [p for p in points if p.timestamp >= now - timedelta(hours=long_h)]
    rate = lambda pts: sum(1 for p in pts if p.escalated) / len(pts) if pts else 0.0
    threshold = baseline * multiplier
    return {
        "firing": rate(short_pts) > threshold and rate(long_pts) > threshold,
        "short_rate": round(rate(short_pts), 3),
        "long_rate": round(rate(long_pts), 3),
        "threshold": round(threshold, 3),
    }
```

混合成本模型通常按 22% 的 AI 转人工升级率做规划，健康的生产平台落在 15%-30% 之间，取决于任务复杂度。[^5] 一个调好的帮助台 agent，在混合复杂度队列上的可持续自动化率更接近 65-75%，对应升级率下限 25-35%。[^6] 这些是起点，不是目标。

## 完成时长：锚定用户，不是锚定 API

“延迟”这个词底下混了两个不同的指标。轮次延迟（turn latency）是从用户发消息到第一个可见 token 出现的时间，对应 OTel 的 `gen_ai.response.time_to_first_chunk`，属于基础设施问题。任务完成时长（task completion time）是从任务开始到出现完成或放弃信号的整段墙钟时间：AI 响应时间，加用户审阅时间，加迭代时间，加下游动作时间。后者才是产品指标。

埋点上的坑在于：把任务完成时长锚在 LLM 调用的时间戳上，而不是用户真正开始任务的时间。一次“按 LLM 锚定算是 45 秒”的支持交互，从用户视角可能是 2 分 15 秒，因为他在发出请求之前花了 90 秒组织语言。检查办法是抽一批 session，把 LLM 锚定的 TTC 和 UI 锚定的 TTC 对比；偏差超过 30% 就说明那个好算的指标不能当代理。

两个参考点锚定了 SLO 的讨论。90% 的客户说快速响应很关键，60% 把“立刻”定义为 10 分钟以内。[^7] AI 介入的支持，解决时间稳定落在 5 分钟以内，而人工队列是 15-45 分钟。如果同一类别里 AI 处理任务的中位 TTC 超过人工处理任务中位 TTC 的两倍，用户就会流向人工渠道，不管 AI 把已经完成的那部分解决得多好。

## 看功能留存，不看产品留存

产品留存看板说这个功能是健康的。并不是。用户回到产品是为了非 AI 功能，第三天之后就没再碰过 AI 助手。这是功能留存的失败，而如果你只在产品层面度量，它完全不可见。

a16z（Andreessen Horowitz）对数百家 AI 公司的分析（2025 年 9 月）发现了一个很有特点的曲线形状：第 0-3 个月先陡降，非核心用户退出，然后对找到可重复价值的用户趋于平缓。[^8] 他们把前期的流失叫作“游客流失”（tourist churn），并建议用 M12/M3（第 12 个月留存除以第 3 个月留存）作为前瞻性的健康信号，因为它把好奇心驱动的下降归一化掉了。

另一个 AI 特有的病症是新鲜感悬崖（novelty cliff）：D1/D7 参与度高得反常，然后等用户发现这个功能只覆盖他实际工作中很窄的一片，就陡降。解法是把埋点从“发生过 AI 交互的 session”换成“发生了成功任务的 session”，并跟踪激活漏斗：第 1、2、3 次 session 里至少完成过一次成功任务的用户占多少？

Amplitude 2025 年覆盖 2600 多家公司的基准发现，从未达到价值里程碑的新用户，超过 98% 在两周内流失。[^9] 含义是：功能留存位于激活的下游。如果你的 B2B 产品 D7 功能留存低于 20%，瓶颈不在模型，而在用户头几次 session 里没有完成过一次成功任务。先修激活漏斗，再优化 prompt。

```python
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum
import statistics

class UserAction(str, Enum):
    ACCEPTED = "accepted"
    REJECTED = "rejected"
    REGENERATED = "regenerated"
    ESCALATED = "escalated"
    ABANDONED = "abandoned"

@dataclass
class SessionRecord:
    session_id: str
    user_action: UserAction
    downstream_action_taken: bool
    task_duration_s: Optional[float]

def compute_metrics(sessions: List[SessionRecord]) -> dict:
    n = len(sessions)
    if n == 0:
        return {}
    n_success = sum(1 for s in sessions if s.downstream_action_taken)
    n_escalated = sum(1 for s in sessions if s.user_action == UserAction.ESCALATED)
    n_resolved = sum(1 for s in sessions
                     if s.downstream_action_taken and s.user_action != UserAction.ESCALATED)
    durations = [s.task_duration_s for s in sessions if s.task_duration_s is not None]
    return {
        "total": n,
        "task_success_rate": round(n_success / n, 3),
        "raw_deflection": round((n - n_escalated) / n, 3),
        "true_resolution": round(n_resolved / n, 3),
        "escalation_rate": round(n_escalated / n, 3),
        "median_ttc_s": round(statistics.median(durations), 1) if durations else None,
    }
```

这五个指标，埋点做对了，才能告诉你 AI 功能到底有没有用。OTel 的 span 告诉你 API 是不是活着。两层都重要；把任何一层当成全貌，就是绿色仪表盘和下滑留存并存的由来。

## 参考文献

[^1]: OpenTelemetry, "GenAI Semantic Conventions: Spans", v1.41.1 (Development), June 2026. https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/

[^2]: NVIDIA, "Mastering Agentic Techniques: AI Agent Evaluation", May 2026. https://developer.nvidia.com/blog/mastering-agentic-techniques-ai-agent-evaluation/

[^3]: Voiceflow, "What Ticket Deflection Rate Actually Means", March 2026. https://www.voiceflow.com/blog/what-ticket-deflection-rate-actually-means

[^4]: Microsoft, "Microsoft Copilot Studio: Deflection Overview", January 2026. https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/deflection-overview

[^5]: Tian Pan, "Escalation Rate Is the Eval Signal Your Offline Tests Missed", May 2026. https://tianpan.co/blog/2026-05-11-escalation-rate-leading-indicator-offline-eval-missed

[^6]: Micheal Lanham, "The 10% Rule: Scale AI Agents to Production", May 31, 2026. https://micheallanham.substack.com/p/the-10-rule-scale-ai-agents-to-production

[^7]: Wayne Diamond / Desku, "Time to Resolution: Definition, Measurement and Reduction", January 2024 (updated June 2026). https://desku.io/helpdesk-glossary/what-is-a-resolved-ticket/

[^8]: Andreessen Horowitz, "Retention Is All You Need", September 2025. https://a16z.com/ai-retention-benchmarks/

[^9]: Digital Applied, "The 2026 SaaS Onboarding Metrics Framework" (citing Amplitude 2025 benchmark, 2,600+ companies), May 2026. https://www.digitalapplied.com/blog/customer-onboarding-time-to-value-2026-saas-metrics-framework
