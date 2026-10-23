---
title: "失败恢复的用户体验"
description: "AI 的失败不是边角情况。设计好回退级联、重试入口和人工交接，让那些你拦不住的失败不至于毁掉信任。"
slug: 01-failure-recovery-ux
part: 11
chapter: 1
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/04-errors-retries-fallbacks
  - part-7-agent-engineering/07-human-in-the-loop
  - 00-ai-ux
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 失败恢复的用户体验

2024 年 2 月，不列颠哥伦比亚省的一家小额索赔法庭判加拿大航空（Air Canada）败诉。他们的客服聊天机器人曾笃定地告诉一位客户，可以购买丧亲优惠票（bereavement fare），事后再申请退款。客户照做了。这项政策根本不存在。加拿大航空辩称，聊天机器人是“独立的法律实体”，应为自己的说法负责。法院不认这个说法。[^1] 那个机器人没有崩溃，没有返回 500，它给出的是一段流畅、自信、完全错误的回答，而一家真实的航空公司为此付了账。

这就是设计问题所在。传统软件失败得很响：一个 500、一段堆栈、一个点不动的按钮。AI 失败得很安静，藏在一段通顺的文字里，系统和用户在当下都察觉不到。信任的衰减是非线性的。2015 年的一项研究（eta-squared = 0.141，“具有相当大的实际意义”）发现：同样严重的一次错误之后，用户对 AI 顾问的信心流失得比对人类顾问更快。[^1] Baymard Institute 在放 AI 生成的检查进入其生产审计工具之前，先定下 95% 的准确率下限，理由是：准确率只有 70% 时，用户分辨不出 10 条建议里哪 3 条是错的，核对的工作量已经超过工具本身的价值。[^1]

本章讲的是：对那些你拦不住的失败，该建什么。供应商层面的重试（retry）机制（指数退避、幂等性、429 与 503 的分类）在[错误、重试与回退](../part-2-working-with-model-apis/04-errors-retries-fallbacks.md)里。审批门与“人工处理后恢复”的 agent 循环机制在[人在回路（Human-in-the-loop）](../part-7-agent-engineering/07-human-in-the-loop.md)里。压在这两者之上的是面向用户的体验：模型不确定时、放弃时、把活交出去时，屏幕上显示什么。

## 回退级联是状态机，不是一串顺序步骤

大多数 AI 功能上线时只有两个状态：能用和坏了。模型调用成功，或者用户看到一个报错。这相当于生产环境里一个没有副本的 Web 服务。取而代之的模式是五级回退级联（fallback cascade）：每一级都是一次更简单的响应，而请求可以在执行途中根据信号逐级下掉。[^2]

- **第 1 级：前沿模型。** Sonnet 4.6、GPT-5.4、Gemini 3.5 Flash。正常路径。
- **第 2 级：同系列的便宜模型。** Haiku、Mini、Lite。成本降 50-70%，质量有可测量的下降，延迟落在 50-100ms 区间。
- **第 3 级：语义缓存命中。** 在调用任何模型之前，先看最近的请求是否足够相似、缓存响应是否够用。命中缓存把延迟从约 1.7 秒降到约 50ms。[^2]
- **第 4 级：确定性回退。** 规则、模板、FAQ 查表。有个团队在这一层用一棵梯度提升树替掉了 12 个深度学习模型的集成；延迟从 200ms 降到 10ms，准确率有明显下降，但故障期间功能一直能用。[^2]
- **第 5 级：升级到人工。** 用于那种“确定性的答案答错还不如没有答案”的决策。

决定何时下掉一级的触发条件有两类。延迟：用逐级超时，不用全局超时。流式响应端到端常常要 30 到 120 秒，一个 5 秒的全局超时会砍掉绝大多数有点分量的生成。错误：每个供应商挂一个三态（关闭、打开、半开）熔断器，最近 60 秒内失败率超过约 20% 就打开，放一个探测请求过去，再决定是否重新闭合。

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
        self.state = "closed"

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "open"

    def is_available(self):
        if self.state == "closed":
            return True
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "half-open"
                return True
            return False
        return True

    def record_success(self):
        self.failure_count = 0
        self.state = "closed"
```

置信度*告诉不了*你的，是什么时候该下掉一级。GPT-4 识别自己推理链中错误的准确率只有 52.9%。[^2] 自报的置信度噪声太大，不足以触发回退。schema 是否合规、关键词是否出现、检索相对预期内容的召回率、校验器的通过率：这些才管用。模型对自己答案的看法不管用。

大多数团队省掉的那项纪律，是让级联保持活着。上线时搭好、之后从未跑过的第 4 级确定性回退，会变成下一次故障里的叠加故障：过期的模板、旧的产品名、上季度的价格。如果生产环境里回退的调用率长期是零，那要么主路径从不失败（很少见），要么熔断器配错了。每季度做混沌测试，在接近真实的负载下刻意触发每一级。把回退测试当成 CI 门禁，而不是下一次事故里的英雄主义。

## 重试不该像一条死路

重试入口（retry affordance）是这样一种 UI 元素：它让用户从同一个任务状态重新唤起 AI，而不用从头再来。这个区别很重要：“重新开始”逼用户把已经给过的上下文再交一遍；重试保留状态，只重跑失败的那一步。对话设计把重试分成三类失败，每类有各自的处理方式。[^3]

- **没有输入。** 用户没有回应。先安静等待；停顿更久再重新搭话；还不行才优雅地结束对话。不要立刻把同一个问题再问一遍。
- **匹配不上。** 输入来了，但映射不到任何意图。第一次重试：给出具体选项澄清（“您是想改签，还是取消？”）。第二次重试：彻底换形式（用按钮，不用自由文本）。第三次重试：收窄的菜单或升级人工。绝不要把同一句提示原样重复三次。
- **误识别。** 模型理解错了意图，还自信地往下走。这是最危险的失败，因为它听起来是对的。解法是无摩擦纠错：“已更正：罗马。您哪天出发？”

最容易把挫败感越滚越大的做法，是把同一句兜底话原样重复。第一次还能忍，第二次就烦，第三次用户走了。每次重试都该换策略，而不是加大音量。另一半在措辞：话要把成本放在系统身上，不是用户身上。“我没听懂”让用户觉得自己有问题。“我不太确定理解对了，您是指 X 还是 Y？”能把人留在对话里。

## 升级到人工是设计好的模式，不是失败

本章最硬的实证信号：86% 的消费者在与聊天机器人交互时希望能转人工，而当看不到任何升级路径时，40% 的人会直接放弃这段对话。[^4] 光是把“转人工”按钮摆出来，就能降低挫败感，哪怕用户根本没点。

把升级当成失败，才是设计上的错误。工程师把 AI 那条路做得一丝不苟，人工交接却在运行时临时凑。结果是接管（override）感觉像系统报错，而不像一个设计内的运行模式。信任流失最多的地方，是自动处理与人工处理之间那道接缝，而不是触发接缝的那次失败。[^5]

四类触发条件基本覆盖了真实产品需要的所有升级：

| 触发类型 | 由什么触发 | 阈值参考 |
|---|---|---|
| 基于置信度 | 模型自评跌破某个下限 | 通用客服 60-70%；企业 80-85%；金融 85% 以上 |
| 基于权限 | 动作超出 AI 的授权范围 | 硬上限，不走置信度判断（例如 500 美元以上的退款） |
| 基于异常 | 检测到循环、token 速度飙升、重复的相同工具调用 | 熔断器形态；5 次以上相同调用或连续 3 次错误 |
| 基于能力 | 任务在结构上就超出范围（不支持的语言、未知领域） | 提前定义为早退规则 |

```python
from enum import Enum
from dataclasses import dataclass

class EscalationReason(Enum):
    CONFIDENCE_LOW = "confidence_low"
    EXPLICIT_REQUEST = "explicit_request"
    PERMISSION_BOUNDARY = "permission_boundary"
    LOOP_DETECTED = "loop_detected"

@dataclass
class EscalationDecision:
    should_escalate: bool
    reason: EscalationReason | None

def should_escalate(confidence: float, tool_calls: int, message: str,
                    request_amount: float = 0.0, ceiling: float = 5000.0) -> EscalationDecision:
    explicit = ["talk to a person", "speak to an agent", "real human"]
    if any(p in message.lower() for p in explicit):
        return EscalationDecision(True, EscalationReason.EXPLICIT_REQUEST)
    if confidence < 0.65:
        return EscalationDecision(True, EscalationReason.CONFIDENCE_LOW)
    if tool_calls > 5:
        return EscalationDecision(True, EscalationReason.LOOP_DETECTED)
    if request_amount > ceiling:
        return EscalationDecision(True, EscalationReason.PERMISSION_BOUNDARY)
    return EscalationDecision(False, None)
```

这些阈值不是玄学。拿带标注的验证集跑模型，把阈值的经验值调到运维团队能承受的误报率上。而且要用真实的模型来校准：同一个模型自报的 logprobs 干这件事噪声太大。

升级的另一半，也是大多数团队做砸的那部分，是交接包本身。

## 失忆式交接

客户花五分钟向机器人讲清自己的问题，然后被接到人工，人工开口第一句是“今天有什么可以帮您？”这就是失败。原因在于 AI 积累的状态（试过什么、用户说过什么、置信度为什么掉下来）没有跨过那道接缝。

Conferbot 公布的交接质量分与结果几乎成线性关系：1.0-2.0 分的交接，平均处理时长 18 分钟，交接后 CSAT 52%；4.0-5.0 分的交接，处理时长 5 分钟，CSAT 92%。[^4] 每提高 1 分，处理时长大约缩短 30%，CSAT 提高 10-15 点。

一份生产可用的交接包带六样东西，并在客户接入之前用 10-15 秒被读完（这叫“暖转接”）：

- **问题摘要。** 一句话，AI 生成，说清客户想要什么、有多急。
- **客户身份。** 姓名、账户 ID、最近订单、终身价值。
- **对话历史。** 带时间戳的完整记录，不要转述。
- **情绪状态。** 升级那一刻的标记，通常带颜色。
- **试过的动作与结果。** “提出退款，客户拒绝，想要换货。”
- **升级原因。** 哪条触发条件命中、为什么；“一封标记为退款的邮件，置信度 0.62”比什么都不说有用。

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class HandoffPackage:
    issue_summary: str
    customer_id: str
    escalation_reason: str
    conversation_history: list[dict]
    actions_attempted: list[str]
    confidence_at_escalation: float
    escalated_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())

    def to_agent_card(self) -> str:
        return "\n".join([
            f"ISSUE: {self.issue_summary}",
            f"CUSTOMER: {self.customer_id}",
            f"REASON: {self.escalation_reason}",
            f"CONFIDENCE: {self.confidence_at_escalation:.0%}",
            f"TRIED: {', '.join(self.actions_attempted) or 'none'}",
            f"TURNS: {len(self.conversation_history)}",
        ])
```

能抓出这套东西到底有没有用的指标是上下文利用率：在所有升级的会话里，人工坐席不需要重问机器人已经问过的东西的比例。目标 90% 以上；低于 70% 说明交接包没人读，或者没带上坐席需要的东西。

对落在 EU AI Act 第 14 条范围内的产品（医疗、就业、信贷、执法、基础设施领域的高风险系统），该条于 2026 年 8 月 2 日生效，人工监督是监管底线：用户必须能够无视、接管或中断 AI 的输出，包括通过一个停止按钮。[^6] 这是最低要求，不是及格线。

## 诚实的空状态与错误状态

大多数 AI 产品上线的样子：屏幕正中一个输入框，上面写着“随便问”。这放弃了所有帮用户第一次就问对的机会。诚实的空状态要说清四件事：这个 AI 在这里能做什么、不能做什么、一个好的开场提示长什么样、它不确定的时候会怎样。

错误状态有同样的诚实问题。明明意思是“我做不到”，却说成“我没听懂”，制造了本可避免的困惑。值得留下的分类是：

- “我没听懂”：真正的意图识别失败。
- “这个我在这里做不了”：能力边界。
- “这件事我没把握帮上忙”：明确承认置信度不足。
- “我这边出了问题”：供应商或基础设施故障。
- “这个请求还缺点信息”：用户自己能补的错误。

这背后的信任账算得很不留情。到 2026 年，超过 60% 的消费者表示对企业如何使用 AI 与自己交互缺乏信心；只有 17% 的人说自己的 AI 体验在变好。[^7] 普华永道（PwC）的消费者洞察调查发现，73% 的客户会在一家失去自己信任的企业显著减少消费，44% 的人在信任崩塌后彻底停止购买。[^8] 当模型的置信度掉到你设定的可靠性下限以下时，把这个状态摆出来（“这个回答基于有限信息”、部分结果、一个主动的“转人工”按钮），挽回的信任比给出一段流畅的错误回答多得多。加拿大航空那个聊天机器人，就是“另一条路要付什么代价”有据可查的案例。

## 参考文献

[^1]: Tian Pan, "The AI Reliability Floor: Why 80% Accurate Is Worse Than No AI at All", April 16, 2026 (updated May 6, 2026). https://tianpan.co/blog/2026-04-16-ai-reliability-floor-trust-threshold

[^2]: Tian Pan, "The Fallback Cascade: Why Your AI Feature Needs Five Failure Modes, Not One", May 2, 2026. https://tianpan.co/blog/2026-05-02-fallback-cascade-graceful-degradation-ai-features

[^3]: Dr. Carmen Martinez, "When the Conversation Breaks: Repair Flows, Fallbacks, and Recovery Strategies", May 21, 2026. https://drcarmenmartinez.substack.com/p/error-handling-ux-writers-conversational-design

[^4]: Conferbot Team, "How to Build a Chatbot That Hands Off to a Human Agent Without Losing Context", October 2025 (updated 2026). https://www.conferbot.com/blog/chatbot-human-handoff-guide

[^5]: Tian Pan, "Human Override as a First-Class Feature: Designing AI Systems That Fail Gracefully to Human Control", May 7, 2026. https://tianpan.co/blog/2026-05-07-human-override-protocols-ai-systems

[^6]: EU AI Act, "Article 14: Human Oversight", Regulation (EU) 2024/1689. Entry into force August 2, 2026. https://artificialintelligenceact.eu/article/14/

[^7]: Wharton / Knowledge@Wharton, "Is AI Killing User Experience?", April 2026. https://knowledge.wharton.upenn.edu/article/is-ai-killing-user-experience/

[^8]: Cindy Rodriguez Constable, "The Missing Variable In Every AI Business Case: Your Customer", Forbes, May 31, 2026. https://www.forbes.com/sites/cindyrodriguezconstable/2026/05/31/the-missing-variable-in-every-ai-business-case-your-customer/
