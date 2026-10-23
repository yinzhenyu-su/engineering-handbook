---
title: "防滥用"
description: "限流、支出上限、按租户的预算，以及在账单上体现出来之前就抓住自动化滥用的那些行为信号。"
slug: 03-abuse-prevention
part: 9
chapter: 3
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/00-calling-models
  - part-2-working-with-model-apis/06-gateways-and-routing
tags: [llm, ai-safety]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 防滥用

2024 年 5 月，Sysdig 威胁研究团队发布了一份事后分析，讲的是一场他们称为 LLMjacking 的真实攻击。攻击者利用受害方 Web 服务器上的一个 Laravel RCE 漏洞，拿到 AWS 凭据，然后在 Bedrock 上跑 Claude 推理。账单由受害方付。规模化之后的极端速率是：单个被攻陷的账号每天超过 $46,000。攻击者还通过一个反向代理转卖访问权限。[^1]

那个数字就是防滥用（abuse prevention）要挡住的东西。在你的网关看来，AI 端点就是普通的 HTTP API，但单位经济账完全不同。一个不加约束的请求可能吃掉 50,000 个 token、几美分的服务商成本。一个循环、一枚泄露的密钥，或者一个精心构造的长上下文请求体，每小时就能变成几千美元，而且不带任何对应收入。驱动这套工程的有三种威胁形态：外部滥用（爬虫、凭据窃取）、内部超额消耗（跑飞的批处理任务、agent 重试循环），以及经济性利用（OWASP 的 LLM10:2025 把它叫作“Denial of Wallet”，钱包拒绝服务）。[^2]

三种威胁用同样的三层来防：服务商限流、应用层按租户的控制、行为层面的滥用检测。每一层拦的是不同类别的攻击，谁也替代不了谁。

## 服务商的限额隔不开你的租户

第一件要内化的事：Anthropic 和 OpenAI 的限流，保护的是*服务商的基础设施不被你所在组织打爆*，而不是你的租户之间互不影响。如果你的某个客户把整个组织共享的 RPM 额度用光，其他所有客户都会看到 429，直到桶重新被填满。

Anthropic 文档里写的算法是令牌桶（token bucket）：容量会被持续补充到你的上限，而不是在固定间隔上重置。[^3] OpenAI 用的是按分钟滚动的窗口，语义同样连续。[^4] 两家服务商在大多数对话端点上都同时约束三个独立维度：每分钟请求数（RPM）、每分钟输入 token 数（ITPM）、每分钟输出 token 数（OTPM）。一个带 100,000 输入 token 的请求能稳稳待在 RPM 限额之内，却把 ITPM 额度打穿。三个维度你都得管。

关于 Anthropic 的限额，有两件运维上的事值得知道（截至 2026 年 6 月）。第一，`cache_read_input_tokens` 不计入 ITPM。一个 ITPM 上限 2M、缓存命中率 80% 的组织，实际上每分钟能处理 10M 总输入 token。[^3] 第二，组织内的工作区（workspace）可以在组织限额之下各自设限，这是防止某个内部团队跑飞拖累别人的主要机制。把工作区限额定在组织上限的 70-80%，留出突发余量。

账户层级（tier）决定每月的支出天花板。Anthropic 从 Tier 1（每月 $500，充 $5 可进入）到 Tier 4（每月 $200,000，充 $400）。OpenAI 的层级从 Tier 1（每月 $100）到 Tier 5（每月 $200,000，累计消费 $1,000）。[^3][^4] 这些天花板是爆炸半径的上限，不是你可以依赖的预算。哪怕在 Tier 1，$500 也远远超过一个免费用户该产生的成本。

## 在网关做按租户的预算

应用层按租户的控制，才是真正的成本隔离所在。最小架构是两件东西：一个能感知 token 的限流器，加一个支出上限（spend cap），都以租户为键。

成本随模型变化。2,000 token 输入配 500 token 的 Claude Sonnet 4.6 输出，截至 2026 年 6 月每次调用 $0.0135（每百万输入 token $3，每百万输出 token $15）。[^5] 一个吵闹的租户每天 1,000 次调用，一个月就是 $405。现在设想攻击者把这租户的 API key 写成脚本在跑。这笔账很快就难看了。所以要设的按租户月预算，是预期用量的合理倍数（付费档 1.5 倍是个不错的默认值；免费档更紧）。

```python
from dataclasses import dataclass

@dataclass
class SpendCapResult:
    allowed: bool
    estimated_cost: float
    remaining_budget: float
    reason: str | None = None

def check_spend_cap(
    current_spend: float,
    monthly_budget: float,
    input_tokens_est: int,
    output_tokens_est: int,
    input_price_per_mtok: float = 3.0,
    output_price_per_mtok: float = 15.0,
) -> SpendCapResult:
    est = (input_tokens_est / 1_000_000 * input_price_per_mtok +
           output_tokens_est / 1_000_000 * output_price_per_mtok)
    remaining = monthly_budget - current_spend
    if remaining <= 0:
        return SpendCapResult(False, est, remaining, "budget_exhausted")
    if est > remaining:
        return SpendCapResult(False, est, remaining, "would_exceed_budget")
    return SpendCapResult(True, est, remaining - est)
```

这段代码包裹的模式是**乐观预留（optimistic reservation）**：用已知的输入 token 数加上配置的 `max_tokens` 估算成本，在 Redis 里对该租户的累计支出计数器做原子自增（对每租户的键执行 `INCRBYFLOAT`），把请求发出去，等响应回来后再拿实际的 `usage.input_tokens` 和 `usage.output_tokens` 对账。调用前的延迟只多一次 Redis 往返，在 LiteLLM 的测量里约 0.8 ms。[^6] 原子性是全部的要点：非原子的读-检查-写循环会让两个并发请求都看到“已花 $0.09，上限 $0.10”，双双通过检查，预算就爆了。

预算被击穿时返回 HTTP 402（Payment Required）。它在语义上是对的，也让客户端能把预算耗尽和限流干净地区分开。LiteLLM 用 400 加 `error.type: "budget_exceeded"`；OpenAI 和 Anthropic 则把预算超限混进 429。[^3][^4][^6]

![一张横向流程图：客户端请求从左侧进入，依次穿过三个纵向堆叠的过滤框，分别标注为行为检测器、应用层限流器与支出上限、服务商限流，最后到达右侧的 LLM 服务商；失败的请求向下退出，错误码为 403、429 和 402，以及服务商侧的 429；每一层都标注了它能拦住的那类攻击](../../../ai-eng/part-9-security-and-governance/images/03-three-layer-stack.png)
*三层，三类要拦的失败。每一层都看得到其他层看不到的东西。*

## 选对限流算法

四种标准的限流算法；对 AI 端点来说，两种对、两种错。

**令牌桶**是默认选择。每个租户一个桶，有最大容量和补充速率。每个请求从桶里消耗一定权重；桶空了就返回 429。空闲时桶持续补充。这样能容纳合法的突发型 AI 负载（用户贴一篇长文档让模型分析）而不惩罚它。Anthropic 用的就是它，这就是行业信号。[^3]

**滑动窗口计数器**是那种“严格的按租户公平比突发容忍更重要”时的务实选择。它每个租户存两个整数计数器加一个窗口时间戳，把当前速率估成加权混合值，内存始终是 O(1)。用在共享的免费档上——那里任何一刻都不该有用户冲过自己那份公平份额。

**滑动窗口日志**是精确版本：把每个请求的时间戳存进 Redis 有序集合，让旧条目过期，数剩下的。只有当你必须做到按身份精确计量、并且能接受内存开销（它会在真正的攻击期间随攻击量一起涨）时才用它。[^7]

**固定窗口**对任何公开的 AI 端点都是错的选择。边界放大这个 bug 早有详细记录：一个脚本在 11:59:59.9 发 N 个请求，在 12:00:00.1 再发 N 个，每一次重置都把有效速率翻一倍。别用。[^7]

只要有不止一个网关实例，分布式协同就成了问题。标准做法：用共享 Redis 集群作唯一事实来源，本地内存缓存按短间隔与它对齐。LiteLLM 每 10 ms 同步一次，相比每次请求都走一遍 Redis 往返能拿到约 2 倍吞吐，代价是三个实例、100 RPS 下最多十个请求的漂移。[^6] 对成本控制来说这点漂移可以接受。如果你要精确计量，那每次检查都走 Redis pipeline（在一个 `MULTI/EXEC` 块里执行 `ZREMRANGEBYSCORE`、`ZADD`、`ZCARD`），并接受那份延迟。

## 限流抓不住的东西

高级的滥用会故意待在限流线以下。爬虫或 API 转卖商按定义会把速率调到刚好低于阈值，抓住它的是行为检测。五个信号，大致按性价比从高到低排：

- **时间规律性。** 人有方差：思考停顿、阅读时间、页面跳转。机器人只有一个 `time.sleep()` 间隔。10 分钟窗口内请求间隔的变异系数（标准差/均值）低于 0.1，对对话型端点就是很强的机器人信号。
- **没有昼夜规律。** 真实用户跟着日曲线走，工作时间或者晚上。一个 7×24 平稳、贴着限流上限跑的密钥，就是教科书式的 LLMjacking 特征。对每个租户的小时级 token 消耗，拿 14 天滚动基线跑 CUSUM 检测。[^1]
- **输出 token 长度异常。** 每个产品都有自己的输出分布。数据外泄攻击和模型抽取尝试会产生异常长的结构化输出。若某会话最近五次输出超过该租户 30 天移动平均的三个标准差，就把它标出来。
- **模型枚举。** Sysdig 的分析抓到攻击者发送 `max_tokens_to_sample: -1` 的 `InvokeModel`，借此枚举可访问的模型又不烧贵的 token。[^1] 同一个身份跨多个模型 ID 连续收到 `ValidationException` 或 `AccessDenied`，就是侦察特征。
- **客户端指纹线索。** OAI Reverse Proxy 的 user-agent 很有辨识度。维护一份黑名单。从不更换 user-agent、缺少真实 SDK 会带的请求头、或者使用非标准 TLS 指纹的客户端，都是信号。

Arcjet 的工程分析（2026 年 2 月）把这件事分成体量控制与行为控制两类：体量控制（限流、支出上限）挡住任何来源的成本失控；行为控制抓体量阈值以下的模式。[^8] 两个都要用。分层方式是让体量控制做硬停，行为控制做异常触发器，逐级升级为软动作：一次 CAPTCHA、进人工审核队列、临时封禁。只凭一个行为信号就硬封，噪声太大。

## 死循环那种情况

最贵的滥用往往来自内部。一个 agent 重试失败的 tool call。重试失败。重试的重试失败。agent 想一想，多生成些 token，再试一次。多数 agent 框架默认都不带按会话的迭代次数或成本上限。

2025 年 11 月一起 LangChain 双 agent 的事故，据报道一个跑飞的循环在 11 天里烧掉了 $47,000。[^9] OWASP 把这种模式编在 LLM10:2025 下。[^2] 修法是两个上限串联，不是一个：最大迭代次数*和*每会话最大支出。只限迭代会漏掉长上下文循环——其中一次迭代就能花掉一百美元。只限支出会漏掉快循环——那种原本迭代次数能更早发现。

LiteLLM 的 Agent Gateway 暴露 `max_iterations` 和 `max_budget_per_session`；LangGraph 暴露 `recursion_limit`。两个都设。任何单个 `session_id` 在 30 分钟滚动窗口里累计超过 50 次 LLM 调用，或者单次会话超过 $5，就告警。这些数字只是起点，按你自己的分布去调。

## 识别凭据被盗

Sysdig 那篇 LLMjacking 事后分析，是已公开材料里对 CloudTrail 日志中的凭据滥用写得最细的一份，运维上的经验可以直接搬过来：[^1]

- 默认开启模型调用日志。Sysdig 发现攻击者会主动检查 `GetModelInvocationLoggingConfiguration`，跳过开了日志的密钥。日志本身就是一种主动威慑。
- 用 IAM SCP 把 `InvokeModel` 限制到已知 IP 段或 VPC 端点，而不是整个互联网。
- 创建账号时就设一个保守的月度支出上限。哪怕在 Tier 1，默认也是 $500；对于永远不该承载生产流量的密钥，把它调低。
- 密钥按季度轮换。绝不提交进版本控制。用开了轮换的 Secrets Manager 或 Vault。
- 监控那些侦察特征：`max_tokens_to_sample: -1`、跨模型 ID 的 `ValidationException` 模式、来源 IP 的突然地理位置跳变。

具体到 Bedrock，AWS 在 2026 年 4 月引入了按 IAM 主体自动归因成本，这意味着如果你做到一租户一 IAM 角色，就不再需要单为归因去建一个 Application Inference Profile（AIP）。[^10] 对多个租户共用一个角色的 SaaS，带自定义标签（`tenant_id`、`project`）的 AIP 仍是推荐做法，而且能直接喂进 Cost Explorer 面板。不过归因不等于强制。AIP 还得配上 CloudWatch 对 `InvokeModel` 调用量的告警，在租户越线时触发 Lambda 动作（停用密钥、通知客户）。

## 网关自建还是买

决策规则大致是：在你有超过三种不同租户类别、或者每月 AI 支出超过 $1,000 之前，自己写 Redis 支撑的限流器和支出上限。过了这条线，遇到下面三种情况之一就去用网关产品（LiteLLM、Bifrost 或商业替代）：你需要按模型×按租户的限额；你是多服务商运营、需要在它们之上做统一治理；或者合规要求你有按请求支出归因的审计日志。LiteLLM 公布的生产部署开销是每请求 3 到 8 ms。[^6] Bifrost 在单节点内存场景报的是 11 微秒。[^11] 两者都远低于任何 LLM 调用的延迟下限，所以选型看的是功能与运维模式，不是性能。

想更全面地了解网关架构，见[网关与路由](../part-2-working-with-model-apis/06-gateways-and-routing.md)。本章的令牌桶与滑动窗口机制，可以顺畅延伸到 HLD 层面的[限流器案例](../../hld/part-8-case-studies/02-rate-limiter.md)。

下一章[PII 与隐私](./04-pii-and-privacy.md)讲的是：网关现在开始大规模记录 prompt 和输出，你拿它们怎么办。

## 参考文献

[^1]: Alessandro Brucato (Sysdig Threat Research Team), "LLMjacking: Stolen Cloud Credentials Used in New AI Attack," Sysdig Blog, 6 May 2024, https://www.sysdig.com/blog/llmjacking-stolen-cloud-credentials-used-in-new-ai-attack

[^2]: OWASP GenAI Security Project, "LLM10:2025 Unbounded Consumption," 2025, https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/

[^3]: Anthropic, "Rate limits," Claude API Documentation, accessed June 2026, https://docs.anthropic.com/en/api/rate-limits

[^4]: OpenAI, "Rate limits," OpenAI API Documentation, accessed June 2026, https://platform.openai.com/docs/guides/rate-limits

[^5]: usagebox.com, "Claude API Billing in 2026: Opus 4.7 / Sonnet 4.6 / Haiku 4.5 Limits," May 2026, https://usagebox.com/articles/api-usage-billing-claude-limits

[^6]: BerriAI, "Budgets, Rate Limits," LiteLLM Proxy Documentation, accessed June 2026, https://docs.litellm.ai/docs/proxy/users

[^7]: Cassie Gatton, "Rate Limiting Algorithms: Token Bucket vs Sliding Window vs Fixed Window," Arcjet Blog, 24 March 2026, https://blog.arcjet.com/rate-limiting-algorithms-token-bucket-vs-sliding-window-vs-fixed-window/

[^8]: Cassie Gatton, "Detecting Bots, Scraping, and AI-driven Abuse at the Application Layer," Arcjet Blog, 10 February 2026, https://blog.arcjet.com/detecting-bots-scraping-and-ai-driven-abuse-at-the-application-layer/

[^9]: Tian Pan, "Token Spend Is a Security Signal Your SOC Isn't Watching," tianpan.co, 23 April 2026, https://tianpan.co/blog/2026-04-23-token-spend-security-signal-soc

[^10]: AWS Machine Learning Blog, "Manage multi-tenant Amazon Bedrock costs using application inference profiles," July 2025; "Introducing granular cost attribution for Amazon Bedrock," April 2026, https://aws.amazon.com/blogs/machine-learning/manage-multi-tenant-amazon-bedrock-costs-using-application-inference-profiles/

[^11]: Kamya Shah, "Budget and Rate Limit Architecture for Multi-Tenant LLM Platforms," Maxim AI / Bifrost Engineering Blog, 3 June 2026, https://www.getmaxim.ai/articles/budget-and-rate-limit-architecture-for-multi-tenant-llm-platforms/
