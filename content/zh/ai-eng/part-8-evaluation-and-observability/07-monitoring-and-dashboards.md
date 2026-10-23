---
title: "监控与仪表盘"
description: "为什么 LLM 系统要把成本、延迟、质量和用量放在同一块屏上；怎么对质量回归和成本尖峰告警；2026 年该选哪家可观测性平台。"
slug: 07-monitoring-and-dashboards
part: 8
chapter: 7
difficulty: intermediate
prerequisites:
  - 05-online-evaluation-and-ab-testing
  - 06-tracing
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 监控与仪表盘

你的 APM 仪表盘（dashboard）全绿。CPU 曲线平直。p99 延迟没动过。错误率为零。而产品其实已经悄悄坏了：过去一周 faithfulness 分数从 0.84 漂到 0.71，每天给 OpenAI 的账单翻了三倍，因为某个人的提示词模板里长出了一段 4,000 token 的 system message，还有三个客户一声不吭就流失了。这些故障，在后端工程师凭本能搭出来的那块屏上，一条都算不上“事故”。

这道缝隙就是本章存在的全部理由。LLM 应用失效的那些维度，传统基础设施监控根本看不见；解法是一块四分屏的仪表盘，外加三类告警——多数团队是踩了坑才学会的。

## 真正要紧的四块面板

一个后端服务大约只有两种故障模式需要仪表盘呈现：变慢了，或者在报错。LLM 应用有四种，而且必须放在同一块屏上，因为诊断的依据是它们*一起*怎么动。

![一块 2×2 的仪表盘网格，展示 LLM 应用的成本、延迟、质量、用量四块面板，面板之间的箭头标出把彼此联系起来的诊断模式](../../../ai-eng/part-8-evaluation-and-observability/images/07-four-pane-dashboard.png)
*成本、延迟、质量、用量放在同一块屏上。诊断价值在跨面板的模式里：成本升而延迟稳，通常意味着提示词变长了；质量降而成本稳，通常意味着厂商悄悄换了模型。*

**成本**是每个时间窗口的总美元支出，按模型、功能、提示词模板，以及（在需要的地方）按用户或租户拆分。算式是 `input_tokens * price_in + output_tokens * price_out`，在 trace 层累加，这样 RAG 流水线的 embedding 花销能和生成花销一起汇总上来。Langfuse 对主流厂商自动算这笔账；LangSmith 则给出了为不支持的厂商通过 metadata 提交自定义成本的做法。[^1][^2]

**延迟**是完整 trace 耗时的 p50、p95、p99，以及流式端点的首 token 时间（time-to-first-token，TTFT）。只看 p50 会骗人。LLM API 的尾部很重，因为单次请求的输出长度能差一个数量级；同样负载下，p99 和 p50 常常相差 5 倍。把 p99 当 SLO，把 p95 当告警阈值，并把 TTFT 记进 OpenTelemetry 的 `gen_ai.response.time_to_first_chunk` 属性。[^3]

**质量**是对生产 trace 抽样后跑自动评审（faithfulness、相关性或任务通过率）得到的滚动均值。每次调用都跑评审会让账单翻倍，所以 5%–10% 采样是可用默认值。聚合窗口要宽到有意义：一个 5 分钟的 p99 至少需要桶里有 100 条 trace，低流量产品应该放宽到一小时。

**用量与使用情况**是请求数、活跃用户数、按类型拆分的错误（rate limit、timeout、content filter），以及 `finish_reason` 的分布。最后这一项被严重低估：如果 `finish_reason: length` 开始飙升，说明上下文窗口快用尽了，回答正在句子中间被截断，而其他任何信号都不会告诉你这件事。

单屏规则不是为了好看，是为了诊断。延迟尖峰同时伴随质量下滑、成本平稳，几乎总是意味着模型路由改了。成本尖峰而延迟没变，几乎总是意味着提示词变长了。质量下滑而成本和延迟都平稳，意味着厂商在你背后悄悄换掉了 `gpt-4o` 的 checkpoint。这些事故，单看任何一块面板都看不出来。

## 每个 LLM 应用都需要三类告警

仪表盘是给人白天浏览的，告警是半夜把人叫起来的。三类就覆盖了值得打电话叫人的故障；其余最多发条 Slack 通知。

**成本尖峰告警**在滚动窗口内的总支出超过基线一定比例时触发。LangSmith 的成本告警把 `input_tokens * rate + output_tokens * rate` 在 5 分钟或 15 分钟窗口上求和，再路由到 PagerDuty、Dynatrace 或任意 webhook。[^4] Langfuse 在 Core 及以上套餐通过 webhook 提供同样的形态。你要抓的两种故障是：失控的 agent 循环十五分钟烧掉一天预算，以及某个 PR 里提示词模板悄悄长出了一段 4,000 token 的前缀。5 到 15 分钟的窗口抓得住循环；每小时汇总抓得住缓慢漂移。

**质量回归告警**在滚动评审分数相对基线下降时触发。真正要紧的设计选择是*绝对阈值还是相对阈值*。绝对告警（“faithfulness < 0.7”）凌晨三点很好理解，但某一类平时就在 0.68 的查询临时占了流量，它就会一直响，团队于是学会了忽略它。相对告警（“当前分数比 7 天滚动均值低两个标准差以上”）扛得住流量结构变化，代价是需要更多历史数据，值班时也更难解释。默认用相对阈值；只有当存在“低于此线产品就不能上线”的硬底线时才用绝对值。

**漂移告警**最难。输入漂移指用户查询的分布变了：出现新的主题簇、一次营销活动带来不一样的人群、竞争对手故障把他们的流量导到你这里。输出漂移指没有发版回答却变了：回答更长、用词不同、语气变了。两者都套不上干净的阈值。生产上的做法是用固定模型对查询做 embedding，在滚动窗口上对 embedding 分布跑 PSI（Population Stability Index）或 KS 检验；更便宜的替代指标是跟踪输出 token 数分布，作为啰嗦程度的信号。Arize Phoenix 内置了 embedding 漂移可视化，这也是真正让它区别于其他产品的那一个功能。

下面是平台无关的逻辑，也就是每家可观测性厂商换个名字实现的同一件事：

```python
"""Cost-spike, latency-spike, and quality-regression alerts."""
from dataclasses import dataclass
import statistics

@dataclass
class TraceWindow:
    cost_usd: float
    latency_p99_ms: float
    quality_scores: list[float]

def check_alerts(
    current: TraceWindow,
    baseline: TraceWindow,
    cost_spike_pct: float = 0.25,
    quality_drop_abs: float = 0.05,
    latency_spike_pct: float = 0.50,
) -> list[str]:
    alerts = []
    if current.cost_usd > baseline.cost_usd * (1 + cost_spike_pct):
        alerts.append(f"COST_SPIKE: {current.cost_usd:.2f} vs {baseline.cost_usd:.2f}")
    if current.latency_p99_ms > baseline.latency_p99_ms * (1 + latency_spike_pct):
        alerts.append(f"LATENCY_P99_SPIKE: {current.latency_p99_ms} ms")
    if current.quality_scores:
        cur_q = statistics.mean(current.quality_scores)
        base_q = statistics.mean(baseline.quality_scores)
        if cur_q < base_q - quality_drop_abs:
            alerts.append(f"QUALITY_REGRESSION: {cur_q:.3f} vs {base_q:.3f}")
    return alerts
```

这些阈值（成本 25%、延迟 50%、质量下降 0.05）是起点，不是行业标准。每个团队都会在第一个月里按自己的流量把它们调一遍，LangSmith 自己的文档也这么直说：先放宽，按观察到的模式收紧，把告警疲劳当成一种真实的故障模式。[^4] 放到架构尺度上，[Eval & observability for AI systems](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md) 讲了这些阈值怎么和 SLO、错误预算挂钩；算式和你对任何服务用的那套一样，就是滚动窗口上的燃尽速率（burn rate）。

> [!WARNING]
> **把模型版本钉死，或者加一条“响应模型变了”的告警。** OpenAI 和 Anthropic 会悄悄给 `gpt-4o`、`claude-sonnet-4-5` 这类模型别名推更新，版本字符串都不带变的。成本不动。延迟不动。质量动了。标准阈值告警不会响。要么钉到带日期的 checkpoint（`gpt-4o-2024-11-20`），要么在 `gen_ai.response.model` 属性和最近 N 次调用不一样时就告警。

## 用一张决策框架看清平台版图

2026 年 LLM 可观测性由四家厂商主导：LangSmith、Langfuse、Arize Phoenix 和 Braintrust。它们的功能列表有 80% 重叠。真正影响采购决策的差异是许可、对 OTel 的友好程度、数据保留期，以及按席位收费还是固定价。下面这张对比才是真正决定选择的依据：

| 平台 | 许可 / 托管 | OTel 接入 | 价格（云） | 默认保留期 | 最擅长 |
|---|---|---|---|---|---|
| **Langfuse** | MIT，免费自托管 | 一等公民的 OTLP | Core 每月 $29 固定价，用户数不限；Pro 每月 $199 | 90 天（Core） | 开源可观测性、eval 工作流、没有按席位抽税 |
| **LangSmith** | 专有；自托管仅企业版 | 接受 OTLP，原生 SDK 为专有 | 每席位每月 $39（Plus）+ trace 超额费用 | 基础 14 天 | LangChain/LangGraph 集成、自主诊断（Engine） |
| **Phoenix (Arize)** | ELv2，免费自托管（不得做 SaaS 转售） | 原生支持，OpenInference 约定 | 开源免费；Arize AX SaaS 每月 $50 起 | 15–30 天（AX） | embedding 漂移可视化、气隙部署 |
| **Braintrust** | 专有，不支持自托管 | 通过 OTLP exporter | Pro 每月 $249 | 14–30 天 | eval 自动化、“loop agent”提示词迭代 |

以上数字均截至 2026 年 6 月，这个市场每个月都在变；签约前去厂商的定价页面核对。[^1][^4][^5][^6]

决策规则并不浪漫。

**默认选 Langfuse Cloud Core（每月 $29），除非你已经住在 LangChain 里。** 它是 MIT 许可，OTel 优先，没有按席位抽税（十人团队和单干开发者付一样的 $29），给 90 天保留期，而且万一以后合规逼你迁走，还有免费的迁移到自托管的路径。2026 年 1 月起它也成了 ClickHouse 的一部分，这让 trace 存储有了可信的长期路线图。[^7] 每月 $199 的 Pro 档是合规外壳（SOC 2、3 年保留、更高的摄取上限）。

**如果已经在大量使用 LangChain 或 LangGraph，就选 LangSmith**，或者你专门想要 Engine：一个每 6 小时跑一次的自主循环，对生产故障做聚类、提出提示词修复方案、生成评审器。对高流量团队来说这确实值真金白银；对一个十人团队来说，在 trace 超额费之前就是每月 $390，相比 Langfuse 的 $29 固定价是实打实的跳档。自托管 LangSmith 需要企业合同；如果数据驻留是硬要求而预算不是，那这就是你要走的路。[^5]

**如果数据不能离开你的 VPC，就选自托管的 Phoenix**，或者你专门需要 embedding 漂移看板。Elastic License 2.0 让你免费跑，没有功能门禁；代价是 ELv2 没拿到 OSI 认证，并且禁止把 Phoenix 作为托管商业服务提供。只认 OSI 许可的团队还是留在 Langfuse 的 MIT 上。[^8]

**把 Braintrust 当补充，而不是主存储。** 它的强项是 eval 自动化：大规模跑闭环的 LLM-as-judge，以及自主迭代提示词的“loop agent”。它在监控上的弱项是保留期：Starter 14 天，Pro 30 天。事故过后 45 天你想做的那次回归分析，这个时长不够。把它和保留期更长的东西一起用。[^6]

关于自托管，再说一句更一般的话。自建可观测性栈胜过云端超额费用的盈亏平衡点大约在每月 500 万条 trace，即便到了那一步，运维负担也是真的：你的应用之外，现在 ClickHouse 升级也得你值班。默认走云端，除非合规逼你，或者你的 trace 量确实大。

## 采样，监控的隐形杀手

有一个实现细节，一旦做错，上面所有东西都会归零。每天 10 万次请求，完整 trace 数据一天就是好几 GB，你不可能无限期存着。多数团队会想到采样，而多数团队采错了类型。

头部采样（head-based sampling）在 trace 开始时决定留还是丢，那时模型还没响应。它便宜、简单。它也会系统性地丢掉那些稀少、复杂、慢的 trace，而质量回归恰恰藏在这些里面。质量告警要么迟到要么根本不响；你抽查采样下来的 trace 看着都健康；同时长尾已经烂了。

尾部采样（tail-based sampling）把 span 缓冲到根 span 结束再决定。更贵，但你可以留下 100% 的错误 trace 和 10% 的成功 trace，这更接近你要的效果。平台支持的话，正确的模式是*按分数触发*的采样：所有 span 都带 metadata 存下来，但只对评审分数低于阈值或 `finish_reason` 异常的 trace 保留完整的 prompt/completion 内容。Langfuse 的客户端脱敏和按分数门控导出能做这件事；其他平台得靠采样规则加保留标签自己拼。

推论是保留策略。LangSmith 基础 trace 默认 14 天是一道悬崖：事故过后 45 天，你要用来诊断它的那条 trace 已经没了。重要的 trace（客户报的问题、A/B 测试样本、评审分偏低的离群点）在创建时就打上延长保留的标签，不要事后补——把已删除的 trace 事后提升保留期是不管用的。

## 接起来之后长什么样

四家平台底下共同的埋点基础，是 OpenTelemetry 的 GenAI 语义约定（semantic conventions），截至 2026 年 6 月版本为 1.41.1，稳定度状态是“Development”。[^3] 只要你的代码按标准发出 `gen_ai.*` 属性，换个 OTLP endpoint 就能换后端，这是每个团队都该提前规划的锁定逃逸路径：

```python
"""Emit OTel GenAI spans to any OTLP backend."""
import os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(
    endpoint=os.environ["OTEL_EXPORTER_OTLP_ENDPOINT"],
    headers={"Authorization": f"Bearer {os.environ['PLATFORM_API_KEY']}"},
)))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("ai-app")

with tracer.start_as_current_span("chat gpt-4o") as span:
    span.set_attribute("gen_ai.operation.name", "chat")
    span.set_attribute("gen_ai.provider.name", "openai")
    span.set_attribute("gen_ai.request.model", "gpt-4o")
    span.set_attribute("gen_ai.usage.input_tokens", 120)
    span.set_attribute("gen_ai.usage.output_tokens", 85)
```

换 endpoint 就换目标：Langfuse 用 `https://cloud.langfuse.com/api/public/otel`，自托管 Phoenix 用 `http://localhost:6006/v1/traces`，LangSmith 用 `https://api.smith.langchain.com/otel/v1/traces`。上面那五个属性喂饱了仪表盘上的每一块面板：token 数驱动成本，span 时长驱动延迟，模型名驱动响应模型变更告警，操作名驱动按功能的拆分。

这套 semconv 还处在“Development”稳定度，意味着属性名在版本之间还可能改。把 OTel SDK 显式钉版本，升级时用 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` 环境变量，让仪表盘查询原子地迁过去，而不是一周后才发现某块面板空了。[^3]

上线前就把仪表盘搭好，别等第一次事故。四块面板、三类告警、钉住响应模型、给 45 天后还会需要的那批 trace 定一套合理的保留策略：这就是 2026 年一个 LLM 应用能用的监控配置。其余都是各家平台的界面细节。

## 参考文献

[^1]: Langfuse, "Pricing," Langfuse Cloud pricing page, June 2026, https://langfuse.com/pricing
[^2]: LangChain, "Cost tracking," LangSmith documentation, https://docs.langchain.com/langsmith/cost-tracking
[^3]: OpenTelemetry Authors, "Semantic conventions for generative client AI spans," OpenTelemetry semconv 1.41.1, June 2026, https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
[^4]: LangChain, "Alerts in LangSmith," LangSmith documentation, June 2026, https://docs.langchain.com/langsmith/alerts-webhook
[^5]: LangChain, "Pricing," LangSmith pricing page, June 2026, https://www.langchain.com/pricing-langsmith
[^6]: Braintrust Data Inc., "Pricing," Braintrust pricing page, June 2026, https://www.braintrust.dev/pricing
[^7]: ClickHouse Inc., "ClickHouse raises $400M Series D and acquires Langfuse," ClickHouse press release, January 16, 2026, https://clickhouse.com/blog/clickhouse-raises-400-million-series-d-acquires-langfuse-launches-postgres
[^8]: Arize AI, "License," Phoenix self-hosting documentation, June 2026, https://arize.com/docs/phoenix/self-hosting/license
