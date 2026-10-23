---
title: "SLO 与事故响应"
description: "AI 故障会带着 HTTP 200 返回垃圾内容。四类 AI 事故（质量回归、注入利用、成本失控、大规模幻觉）、检测它们的语义 SLI，以及把事故关进笼子的应急手册。"
slug: 06-slos-and-incident-response
part: 10
chapter: 6
difficulty: intermediate
prerequisites:
  - part-8-evaluation-and-observability/07-monitoring-and-dashboards
  - part-8-evaluation-and-observability/06-tracing
  - part-8-evaluation-and-observability/01-llm-as-judge
  - 05-shipping-changes
tags: [llm, deployment, cost, slo]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# SLO 与事故响应

[上一章](05-shipping-changes.md)把 2025 年 4 月 GPT-4o 谄媚事件当成一次发布事故来讲：没有金丝雀（canary），回滚花了四天，5 亿周活用户暴露在影响之下。把同样这四天当成一次监控事故（incident）来读，情况更糟。服务商的仪表盘全程绿灯：HTTP 200、延迟正常、零错误，而模型却在为那些本该被反驳的想法大加赞赏，并“偏向于给出过度支持但并不真诚的回答”。[^1] 真正说明出问题的信号，来自社交媒体上的截图。

这就是 AI 事故响应的核心难题。每一个经典 SLI（可用性、错误率、延迟）都在报绿，而主要的故障模式正在造成真实伤害。一个服务可以在 P99 低于 200ms 的情况下返回 HTTP 200，同时又在编造事实、通过被提示词注入的 agent 窃取数据、因为工具调用死循环烧出 5 万美元账单，或者系统性地为危险决策背书。传统 SRE 手册是为确定性系统写的，在那类系统里，执行成功和输出正确是同一件事。对 LLM 来说，两者不是。

本章讲四种不会出现在你 APM 里的 AI 故障模式、检测它们所需的 SLI，以及每种故障发生后第一个小时该走的应急手册（runbook）。

## 你的监控里缺了什么

传统 SLI 衡量服务有没有回应，语义 SLI 衡量服务说了什么。生产环境里最重要的两个语义 SLI：

- **正确率。** 通过自动语义评估（LLM-as-judge、评分标准校验、对照检索到的上下文做接地校验）的回答占比。为控制成本，只在抽样上计算（流量的 5-10%）。它是你的首要质量 SLI。
- **安全合规率。** 满足策略约束的回答占比（不泄露 PII、不含有害内容、不编造法律或医疗结论）。违规是灾难性的，所以预算要严得多（0.1% 或更低）。

常用的补充指标：幻觉率（含无依据事实主张的回答占比）、schema 有效率（针对结构化输出）、工作流完成率（针对 agent）。一个 10 步的 agent，单步成功率 90%，端到端完成率只有约 35%；要盯端到端数字，不是单步。

Google SRE Workbook 里的燃尽率（burn-rate）告警可以直接套用。多窗口、多燃尽率：1 小时内 14.4 倍燃尽率的快燃（fast-burn）触发值班（on-call）呼叫；3 天内 1 倍燃尽率的慢燃（slow-burn）开一张工单。算法和可用性预算完全一样，只是作用在质量 SLI 上。正确率 48 小时内从 92% 掉到 88%，可能只是一张慢燃工单；同样的跌幅如果发生在 2 小时内，就该呼叫了。

两派实践者在阈值风格上存在分歧：

- **绝对阈值**（正确率 < 0.85）。值班时更容易推理，也更容易解释。但查询分布一变就会大量误报，因为更难的问题本来就会拉低分数。
- **相对滚动基线的跌幅**（比 7 日均值低 5 个百分点以上）。对分布漂移更稳健，但前提是先有一个稳定基线，且窗口选得对。

通用质量默认用相对跌幅。安全场景用绝对阈值，因为一次违规就是法律责任。Air Canada 被裁定必须赔偿一名乘客：它的聊天机器人编造了一条丧亲票价政策，而仲裁庭把 Air Canada 提出的“聊天机器人是独立法律实体、对自己的行为负责”的说法称为“一份令人惊讶的陈述”并予以驳回。[^2] 部署方永远是责任方。

不要在窗口内样本少于 100 条的情况下对质量 SLI 呼叫。统计噪声会造出虚假告警，把团队训练成无视告警。低流量产品就把窗口拉长。

## 故障模式 1：质量回归

一次模型更新（应用层的提示词改动，或者服务商侧对基础模型的静默更新）造成回答质量可测量的下滑。返回 HTTP 200、合法 JSON、延迟正常，没有任何基础设施信号。

GPT-4o 谄媚事件是标准案例，OpenAI 自己的事后复盘把根因定为 RLHF 的奖励劫持（reward hacking）：“我们在短期反馈上关注过多”，而对长期用户满意度权重不够。[^1] 模型学到的是优化即时正反馈，不是把话说对。

可靠的早期检测机制是拿固定的黄金测试集做定时金丝雀评估。[^3] 当服务商静默更新基础模型时，金丝雀分数会在用户侧指标变化之前先动。

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class CanaryCase:
    prompt: str
    expected_keywords: list[str]
    label: str

def run_canary_suite(
    call_llm: Callable[[str], str],
    cases: list[CanaryCase],
    baseline_pass_rate: float,
    alert_threshold: float = 0.05,
) -> dict:
    """Run a fixed canary suite and compare to baseline. Page if drop > threshold."""
    results = [
        all(kw.lower() in call_llm(c.prompt).lower() for kw in c.expected_keywords)
        for c in cases
    ]
    rate = sum(results) / len(results) if results else 1.0
    delta = baseline_pass_rate - rate
    return {"pass_rate": rate, "baseline": baseline_pass_rate,
            "delta": delta, "should_alert": delta > alert_threshold}
```

应急手册：

1. **检测。** 金丝雀通过率比基线低 5 个百分点以上，或质量 SLI 的燃尽率超过快燃阈值。
2. **定位。** 检查近期 trace 里的 `gen_ai.response.model` 是否与预期版本一致。对比提示词版本。查检索索引的更新日志。查 embedding 模型版本。
3. **止血。** 如果怀疑是提示词改动，通过提示词管理系统回滚。API 支持的话就钉住（pin）到上一个模型版本；否则切到备用服务商。
4. **验证。** 对回滚后的系统重跑金丝雀，确认通过率回到基线。
5. **加入黄金集。** 失败的用例变成永久回归测试。再加一道发布门禁：金丝雀通过率低于基线 2 个百分点以上就阻止上线。

## 故障模式 2：提示词注入攻击

OWASP 把提示词注入列为 LLM01:2025，LLM 应用的头号安全风险。[^4] 架构层面的成因：LLM 把可信指令（system prompt）和不可信数据（用户输入、检索到的文档、工具输出）放在同一条 token 流里处理，两者之间没有任何边界。

直接注入（攻击者把覆盖指令敲进用户输入）是比较好防的一类。间接注入（指令藏在 AI agent 检索到的内容里：一个 PDF、一封邮件、一条数据库记录）是 2025-2026 年的主流攻击形态，因为载荷是通过可信路径进来的。agent 化会放大危害：一条注入指令可以借助 agent 手里的工具触发真实世界的动作。

生产环境的标准案例是 EchoLeak（CVE-2025-32711），由 Aim Security 在 2025 年 1 月披露，微软 2025 年 5 月修复。[^5] CVSS 9.3。一封精心构造的邮件让 Microsoft 365 Copilot 处理其中的嵌入指令并外泄企业数据，全程零用户交互。载荷串联了分类器绕过、引用式 Markdown 链接操纵和 CSP 代理滥用。

第一个小时的应急手册，目标是减少正在发生的伤害并保住证据：

1. **检测。** 很少被调用的工具出现异常的工具调用频率；来自单一来源的治理策略违规率激增；检索结果里出现已知的注入语法（“ignore previous instructions”）；输出内容与 agent 声明的职责范围不符。
2. **止血。** 从结构化日志里定位受影响的组件和检索来源。只关掉那条具体路由或那个 MCP server，不要整体停服。轮换与受污染组件相关的 API key。启用应急策略规则（拦截指令覆盖、收紧输出 schema、加严日志）。
3. **调查。** 用 session ID 从 SIEM 日志还原攻击时间线。追检索链路，找出是哪份文档送进了载荷。检查受污染窗口内的输出有没有外泄迹象（外连 URL、base64 编码内容、调用攻击者控制端点的工具调用）。
4. **修复。** 在 system prompt 里写清有边界的角色定义。针对该攻击路径更新治理策略。清洗或隔离被投毒的检索源。
5. **验证。** 先在 staging 用同一条攻击向量打补丁后的系统，通过后再在生产开回来。

攻击分类与防御架构（双 LLM 模式、防护栏的放置位置）见 [第 9 部分](../part-9-security-and-governance/00-prompt-injection.md)的安全章节。本章只把注入当作一个事故响应场景来讲。

## 故障模式 3：成本失控

在一份流传很广的实践者记述里，一家金融科技初创公司的多 agent 成本追踪系统跑了 11 天没人发现。Agent A 找 Agent B 要澄清，Agent B 又让 Agent A 解释自己的回答。两边都没有打断循环的逻辑。每周 127 美元的账单变成了 47,000 美元。[^3] 没抛异常、没报警、延迟正常。

服务商侧的账单上限是最后一道兜底，不是纵深防御。OpenAI 的分级额度（Tier 1 每月上限 100 美元，一直到 Tier 5 每月 20 万美元）和项目级控制，有记录显示在部分配置下只起通知作用，并非硬停。成本仪表盘滞后几小时到几天。等你看到尖峰，钱已经花出去了。

防线在代码里，在 agent 编排层，有三道护栏：

- **每个会话的硬性 token 预算**，在你自己的代码里记账，不依赖服务商限额。
- **每个 agent 会话的迭代上限。** 超过 20 次工具调用就杀掉并升级处理。
- **动作去重。** 对工具名加参数做哈希；同一个哈希在一个会话里出现两次，就当成循环信号。

```python
from dataclasses import dataclass, field

@dataclass
class TokenBudget:
    """Per-session circuit breaker for LLM cost control."""
    max_input_tokens: int = 500_000
    max_output_tokens: int = 100_000
    max_iterations: int = 20
    _input_used: int = 0
    _output_used: int = 0
    _iterations: int = 0
    _seen_hashes: set = field(default_factory=set)

    def record_call(self, input_tokens: int, output_tokens: int,
                    action_hash: str | None = None) -> None:
        self._iterations += 1
        self._input_used += input_tokens
        self._output_used += output_tokens

        if self._iterations > self.max_iterations:
            raise RuntimeError("Agent loop exceeded iteration cap; session killed.")
        if self._input_used > self.max_input_tokens:
            raise RuntimeError("Session token budget exhausted.")
        if action_hash is not None:
            if action_hash in self._seen_hashes:
                raise RuntimeError("Duplicate action hash; agent may be looping.")
            self._seen_hashes.add(action_hash)
```

关键细节：熔断开关必须跑在 agent 自己的推理路径之外。在一份公开报道的 OpenClaw 事故里，一个 agent 删掉了 10,000 封邮件，同时无视停止命令，因为停止机制只是提示词层面的一条指令。[^3] 一个困惑、打转或被劫持的 agent 不会可靠地解析“stop”。熔断开关要放在编排层（撤销工具权限、在框架层面终止会话）或 API 网关，不是放在提示词里。Google 的 AI Operator 把所有工具调用都交给一个集中式的 Actuation Agent 做飞行前安全检查；Actuation Agent 与推理引擎在结构上是分离的。[^6]

应急手册：

1. **检测。** 单会话花费估算越过阈值；迭代计数超过上限；动作去重被触发；服务商仪表盘在 15 分钟窗口内涨幅超过 25%。
2. **止血。** 撤销受影响会话的工具权限。停掉排队中的任务。锁住发布流水线。
3. **调查。** 翻会话 trace，找参数完全相同的重复工具调用。找出入口（通常是 agent 之间的交接 schema 不匹配）。确认循环是真实用户操作触发的，还是定时任务触发的。
4. **修复。** 三道护栏缺哪道补哪道。加一个单会话预算守卫。加一个基于成本的熔断器。
5. **加金丝雀。** 一个故意用很低 token 数触发循环条件的测试用例，用来验证熔断器会正确动作。

## 故障模式 4：大规模幻觉

模型生成事实错误但听起来可信的内容，在被人发现之前已经发给大量用户。FAccT 2024 一项针对 Whisper（OpenAI 的语音转文字模型）的研究发现，约 1% 的转写里含有整段从未被说过的幻觉短语，其中 38% 的幻觉带有明确的伤害。[^7] 2025 年一项覆盖 90 个 AI 应用、300 万条用户评论的研究发现，在已被标记为涉及 AI 错误的评论中，约 1.75% 报告了用户遇到的幻觉。[^8] 这些故障没有一个产生系统级错误。

这是最难自动捕捉的故障，因为模型的本职就是生成听起来可信的文本。实践中有效的检测手段：

- **接地校验**，用于以 RAG 为据的系统。校验输出里的每条事实主张能否追溯回一份检索到的文档。这是最可靠的自动化方法。
- **人工改写率。** 如果下游有人审，统计审核者拒收或大改 AI 输出的频率。改写率上升是高保真的领先指标。
- **已知事实答案的金丝雀提示词**（和捕捉质量回归是同一套机器）。
- **置信度分数分布漂移。** 输出越来越不一致或置信度变低的模型，可能正在遇到不熟悉的输入模式。

这份应急手册与质量回归有一处重要不同：止血步骤必须考虑爆炸半径（故障影响面）。医疗、法律、金融领域的高危事故，你可能需要评估是否要做用户通知或主动更正。

1. **检测。** 接地校验失败率超过阈值；人工改写率激增；金丝雀通过率下滑；客服渠道出现用户投诉聚集。
2. **评估爆炸半径。** 可能有多少用户收到了幻觉内容，涉及哪些查询。
3. **止血。** 启用优雅降级：退到只走 RAG、严格接地的模式；加免责声明；高风险场景强制人工复核。
4. **调查。** 检查失败 trace 里的检索上下文质量、提示词版本变更、服务商模型版本变更。
5. **修复。** 加强提示词里的接地约束。加一道检索质量门禁（top-k 检索分数低于阈值就拒答）。收紧事实核查。
6. **沟通。** 受监管领域的高危事故，按法规要求通知受影响的用户。

## 最小 SLI 集合

任何 AI 功能上线生产之前，三个 SLI 是必选项：

- **正确率或任务完成率**（捕捉质量回归和大规模幻觉）。
- **安全合规率**（捕捉策略违规和注入驱动的输出）。
- **单会话成本上限**（捕捉成本失控）。

三者缺一，你就至少漏掉四种故障模式中的一种。让这些指标在实践中可测的仪表盘、告警和链路追踪基础设施，见[监控与仪表盘](../part-8-evaluation-and-observability/07-monitoring-and-dashboards.md)。

四种故障模式共有的规律：传统 SRE 工具报绿，而真实伤害正在发生。质量不是基础设施。模型返回了回答，不等于模型返回了*好*的回答。只要你所在的领域可能造成真实世界伤害、你的 agent 对外部系统有写权限、你的检索语料包含外部内容，或者你的日成本可能被一个循环显著推高，就该对质量呼叫，而不只是对可用性呼叫。这覆盖了大多数生产 AI 产品。

## 参考文献

[^1]: OpenAI, "Sycophancy in GPT-4o: what happened and what we're doing about it", April 29, 2025, https://openai.com/index/sycophancy-in-gpt-4o/
[^2]: Civil Resolution Tribunal of British Columbia, "Moffatt v. Air Canada", 2024 BCCRT 149, February 2024, https://decisions.civilresolutionbc.ca/crt/crtd/en/525448/1/document.do
[^3]: Tian Pan, "Production AI Incident Response: When Your Agent Goes Wrong at 3am", TianPan.co, April 10, 2026, https://tianpan.co/blog/2026/04/10/production-ai-incident-response-runbook
[^4]: OWASP, "LLM01:2025 Prompt Injection", OWASP Top 10 for LLM Applications, 2025, https://genai.owasp.org/llmrisk/llm01-prompt-injection/
[^5]: Aim Security, "EchoLeak (CVE-2025-32711): Zero-click Prompt Injection in Microsoft 365 Copilot", June 2025, https://www.aim.security/echoleak
[^6]: Google SRE, "AI Engineering Reliable Operations", June 2026, https://sre.google/resources/practices-and-processes/ai-engineering-reliable-operations/
[^7]: Allison Koenecke, Anna Seo Gyeong Choi, Katelyn X. Mei, Hilke Schellmann, Mona Sloane, "Careless Whisper: Speech-to-Text Hallucination Harms", ACM FAccT 2024, https://arxiv.org/abs/2402.08021
[^8]: Massenon et al., "'My AI is Lying to Me': User-reported LLM hallucinations in AI mobile apps reviews", Scientific Reports 15:30397, August 2025, https://www.nature.com/articles/s41598-025-15416-8
