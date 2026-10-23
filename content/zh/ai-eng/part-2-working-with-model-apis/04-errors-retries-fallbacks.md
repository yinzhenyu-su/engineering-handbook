---
title: "错误、重试与回退"
description: "如何应对每个 LLM API 都会抛给你的超时、限流与过载响应，配上一套正确的退避、回退与幂等规则。"
slug: 04-errors-retries-fallbacks
part: 2
chapter: 4
difficulty: intermediate
prerequisites:
  - part-0-start-here/01-the-software-baseline
  - 00-calling-models
tags: [llm, http-api, resilience]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 错误、重试与回退

你的客服机器人 14:03 UTC 崩了，因为 Anthropic 返回了一个 529。代码捕获异常，等 0.5 秒，重试，又是一个 529；再等 1 秒，重试，还是 529。十分钟后供应商恢复了，但你的队列已经积了一万两千个请求，每个客户都盯着一个转圈的加载图标。与此同时告警电话响了——因为有人从某篇 REST 教程里抄了个 401 重试循环，而你的账单面板显示，那把一小时前就被吊销的 API key 上已经烧掉了 4,000 美元。

那一段里的每个问题都有解，而且没有一个解法算得上聪明。它们只是一小组规则：哪些错误值得重试、重试之间怎么等、什么时候该切到另一个模型，以及怎么让一次超时不会为同一个答案收你两次钱。

## 先读状态码，再动手

大多数重试 bug 都源于把 LLM 错误当成一个通用异常处理。HTTP 状态码就是全部决策依据，而两家大供应商的差异恰好落在要紧的地方：

| 状态码 | OpenAI 含义 | Anthropic 含义 | 是否重试 |
|---|---|---|---|
| 400 | 请求体有误 | 请求无效 | **不重试。** 修 payload。 |
| 401 | API key 有误 | API key 有误 | **不重试。** 修凭证。 |
| 403 | key 缺少权限 | key 缺少权限 | **不重试。** |
| 408 | 请求超时 | （用 504 代替） | 重试 |
| 413 | （少见） | 请求体超过 32 MB | **不重试。** 把 payload 缩小。 |
| 429 | 你的配额用满（RPM/TPM） | 你的账户被限流 | 重试，按 `Retry-After` |
| 500 | 供应商自己的 bug | Anthropic 内部错误 | 重试 |
| 503 | "Engine overloaded" 或 "Slow Down" | （少见） | 重试 |
| 504 | （少见） | 生成超时 | 重试，或改用流式 |
| 529 | （不使用） | **全基础设施过载** | 重试，但考虑回退 |

那张表里有两行最容易咬人。**401 不能重试。** 你的 key 是错的；睡一觉再试只会一直烧配额，直到监控发现。从通用 REST 教程里抄来的 `@retry(exceptions=Exception)` 这个模式，是“为什么我们的账单半夜暴涨”最常见的单一原因。

另一行是 Anthropic 的 529 与 429 之差。429 表示你的账户撞到了每分钟限额，正确做法是按 `retry-after` 的值等完再试。529 表示 **Anthropic 自己对所有人都过载了**，在同一家供应商上多等没用。[^1] 两者返回的错误信封长得很像，开发者不读状态码，直接按“4xx 那类，重试”来模式匹配。它们是两个不同的问题，解法也不同。

你真正想要的可重试集合，写成代码是：

```python
RETRYABLE_STATUS_CODES = {408, 409, 429, 500, 502, 503, 504, 529}
FATAL_STATUS_CODES = {400, 401, 403, 404, 413, 422}

def should_retry(status_code: int) -> bool:
    return status_code in RETRYABLE_STATUS_CODES
```

如果某个状态码不在可重试集合里，现在就抛出去。致命错误暴露得越快，代价越小。

## 别只是重试，要把重试摊开

假设你的一百台服务器在正午整点同时撞上同一个 429。最朴素的做法是指数退避：睡 `0.5 * 2^attempt` 秒再试。于是这一百台在 12:00:00.5 一起醒来，齐步走地砸向供应商。供应商再 429 一次。它们把等待时间翻倍，又在 12:00:01.5 一起砸过去。这就是惊群（thundering herd），也正是五秒的故障被拉成五分钟的方式。

解法是加随机性。Marc Brooker 在 AWS 做的那次模拟是退避算法的标准参考，它拿四种方案对一百个竞争客户端做了对比，发现 **full jitter（全抖动）** 相比朴素指数退避把客户端的总工作量砍掉一半以上，完成时间却没有变差。[^2] 公式只有一行：

```python
import random
import time

def full_jitter_backoff(attempt: int, base: float = 0.5, cap: float = 60.0) -> float:
    """Sleep up to min(cap, base * 2^attempt) seconds, uniformly at random."""
    return random.uniform(0, min(cap, base * (2 ** attempt)))
```

关键在于上界仍然指数增长（所以长时间故障会等到更久），但实际睡多久是从 0 到上界之间的均匀抽样。一百个客户端现在散落在六秒的窗口里醒来，而不是同一瞬间。

![两条并排的散点图，共用同一时间轴：左边十次重试在 1s、2s、4s、8s 处挤成紧密的竖带；右边同样的十次重试均匀散在整个窗口里](../../../ai-eng/part-2-working-with-model-apis/images/04-jitter-vs-no-jitter.png)
*同样的重试预算，两种策略。朴素指数退避（左）制造同步的重试尖峰；full jitter（右）把负载摊到整个窗口。*

不过在算这个退避之前，先看响应头。当答案本来就知道的时候，两家供应商都会直接告诉你该等多久。OpenAI 发 `retry-after-ms`（毫秒，非标准）或 `Retry-After`（秒）；Anthropic 在 429 上发 `retry-after`。[^3][^4] 有供应商给的提示就用它，没有才回落到你自己的抖动计算：

```python
def wait_before_retry(headers: dict, attempt: int) -> None:
    raw_ms = headers.get("retry-after-ms")
    raw_s = headers.get("Retry-After") or headers.get("retry-after")
    if raw_ms is not None:
        time.sleep(float(raw_ms) / 1000.0)
        return
    if raw_s is not None:
        try:
            time.sleep(min(float(raw_s), 60.0))
            return
        except ValueError:
            pass
    time.sleep(full_jitter_backoff(attempt))
```

把响应头里的值封顶在 60 秒。供应商让你等十分钟，其实是在告诉你该走回退了，而不是让你的 worker 阻塞那么久。

## 这些事 SDK 已经做了大半

上面那个循环你大概不用自己写。`openai` 和 `anthropic` 的 Python SDK 默认都是 `max_retries=2`（算上原始请求共三次尝试），用带抖动的有上限指数退避，尊重 `Retry-After`，并且会加一个 `X-Stainless-Idempotency-Key` 头，让供应商能对重试去重。[^3][^5] 面向用户的交互式调用，默认值就是对的。

只有两种情况你要改它：

- **非交互式批处理**：延迟容忍度高，把活干完比快速失败更重要。把 `max_retries` 提到 5。
- **你把 SDK 放在自己的网关后面**（LiteLLM Router、自研重试层）。把 SDK 客户端的 `max_retries=0`，避免双层重试。这一条你一定会忘。两层重试叠在一起，三次尝试变九次；一条五层深的调用链每层重试三次，本来一个操作能扇出 243 个请求。[^6] **只在整个栈的一层重试，而且让它是最外层。**

## 回退：重试同一家不再有用的时候

重试的前提是供应商很快会恢复。有时候它不会。Anthropic 的 529 影响的是全世界每一个 Anthropic 客户，多睡没用。升级手段是整体切到另一个模型。

回退有两种，它们不是一回事：

- **同一模型，换供应商或区域。** Anthropic Direct 过载时，改在 AWS Bedrock 上跑 Claude Sonnet。权重是同一份，输出质量完全一样。这是安全的默认做法，也应该是你的第一层回退。
- **降级模型。** 从大模型切到更小、更便宜、更快的模型。输出质量会掉。这笔交易是“差答案好过没答案”。

第二种的风险是第一种没有的。小模型写得更短，指令遵循更不可靠，还可能吐出下游代码直接拒收的 JSON 形状。**只有当响应和主模型走同一套校验时，跨模型回退才安全**：一个 Pydantic schema、一份结构化输出规范，或者一个能抓出畸形输出的 eval。没有校验，你就是悄悄用十分钟故障换了一周的、谁都没法复现的隐性 bug。

实践中你不手写这些，而是通过 router 接起来。LiteLLM 是开源里的标准做法：

```python
from litellm import Router

model_list = [
    {"model_name": "primary",
     "litellm_params": {"model": "anthropic/claude-sonnet-4-20250514"}},
    {"model_name": "primary-bedrock",
     "litellm_params": {"model": "bedrock/anthropic.claude-sonnet-4"}},
    {"model_name": "fallback-cheap",
     "litellm_params": {"model": "openai/gpt-4o-mini"}},
]

router = Router(
    model_list=model_list,
    fallbacks=[{"primary": ["primary-bedrock", "fallback-cheap"]}],
    num_retries=2,
    cooldown_time=5,
    allowed_fails=3,
)
```

几个值得弄明白的默认值。LiteLLM 先对主部署试 `num_retries` 次，然后才走回退链。某个部署返回 429，或者一分钟内失败三次，就会被冷却五秒，冷却期内 router 不再往它发流量。[^7] 冷却按部署算，不按模型名算，所以同一模型的两个 Bedrock 区域可以各自失败，不会把整组拖下线。

[网关与路由](06-gateways-and-routing.md) 那一章讲生产环境的 router 配置讲得更深；眼下记住规则：任何跨模型回退之前，先列一个同模型回退；不管响应由哪个部署服务，都按同一套方式校验。

放到架构层面，[多供应商 AI 系统设计](../../hld/part-8-case-studies/37-model-router-gateway.md) 讲了供应商故障切换与 SLA 的白板视角。

## 幂等性：LLM 特有的坑

这里有个在普通 API 工作里不存在的故障模式。你的客户端 30 秒超时。供应商那边在第 31 秒把响应生成完了，并且成功从你账户扣了费。你的代码重试。供应商重跑 prompt，这次采样到不同的 token，返回另一个答案。你现在为两次 completion 付了钱；如果这个 LLM 当时在决定要不要发一封邮件或扣一笔信用卡，**两个决定都已经执行了**。

这打破了重试安全代码背后的标准假设。普通 REST API 是确定性的：重试 `GET /users/42` 返回同一个用户。LLM completion 没有这个性质。重试就是重新采样，重复的重试产生一份重复但不同的输出。[^8]

OpenAI SDK 帮你处理了一部分。它为每个请求生成一个 UUID，在这个请求的每次重试尝试上都以 `X-Stainless-Idempotency-Key` 发出，于是 OpenAI 服务端能认出“这是重试，返回缓存结果”，而不是再跑一次生成。[^3] 如果重试是 SDK 在做的，你在供应商这一层就被覆盖住了。

当 LLM 的输出会触发现实世界的副作用时，这还不够。SDK 的幂等键在 SDK 调用返回的那一刻就失效了；如果你的代码在收到响应之后、持久化之前崩了，下一次运行会带着新 key 发起一个全新请求，副作用执行两次。解法就是 Stripe 处理支付用的那套幂等模式，搬到你的应用层：

```python
import uuid

def generate_with_side_effect(prompt: str, store) -> dict:
    op_id = uuid.uuid4().hex
    cached = store.get(op_id)
    if cached:
        return cached  # 已经跑过了；返回原来的答案

    response = client.chat.completions.create(
        model="gpt-5.4",
        messages=[{"role": "user", "content": prompt}],
        extra_headers={"Idempotency-Key": op_id},
    )
    payload = {"op_id": op_id, "content": response.choices[0].message.content}
    store.put(op_id, payload)   # 在副作用之前持久化
    execute_side_effect(payload)
    return payload
```

两个不显然的点。`op_id` 是给**逻辑操作**（“给 42 号用户发欢迎邮件”）生成的，不是按 HTTP 调用生成，所以进程崩了重启还能找到同一个 key。另外你在触发副作用**之前**持久化 LLM 响应，不是之后，这样调用和副作用之间崩掉，你手上留的是一条可恢复的记录，而不是一个付了钱却拿不到的答案。

流式响应不一样。流一旦开始，你在应用层就没法去重了；部分输出已经离开供应商。对于幂等性比首 token 时间更重要的 agent 步骤，用非流式 API 或 Batch API。流式留给面向用户的渲染层——那里重复生成是成本上的浪费，不是正确性 bug。

## 各项参数设成多少，设在哪一层

按调用类型给一套不会害你的默认值：

- **交互式用户调用。** 用 SDK 默认值。`max_retries=2`、带抖动退避、尊重 `Retry-After`。不要再叠第二层重试。
- **批处理或流水线调用。** 把 SDK 的 `max_retries=5`，但要盯住配额：对一个配额已耗尽的账户重试五次，只是失败得慢一点。
- **带副作用的 agent 步骤。** 应用层幂等键、先持久化再执行，**绝不在决定是否执行的那个循环里重试。** 重试的是 LLM 调用；LLM 调用重试完还是失败，就让这个 agent 步骤本身失败。
- **多供应商路由。** 关掉 SDK 重试（`max_retries=0`），让 router（LiteLLM 或你自己的）统一处理重试与回退。

每次失败时你真正需要记进日志的错误信封是这几项：HTTP 状态码、供应商的错误类型字符串、`Retry-After` 头（如果有），以及这个请求的幂等键。缺了这四个字段，凌晨三点你分不清是重试风暴还是真的故障，剩下的复盘时间都靠猜。

## 参考文献

[^1]: Anthropic，"Errors"，Anthropic API Reference，2026 年 6 月访问。 https://docs.anthropic.com/en/api/errors

[^2]: Marc Brooker，"Exponential Backoff And Jitter"，AWS Architecture Blog，2015 年 3 月（2023 年 5 月更新）。 https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

[^3]: OpenAI Python SDK，"Retries" 文档，由 openai/openai-python 生成（2026 年 3 月提交）。 https://openai-openai-python-73.mintlify.app/concepts/retries

[^4]: Anthropic Support，"Our approach to rate limits for the Claude API"，2026 年 6 月访问。 https://support.anthropic.com/en/articles/8243635-our-approach-to-api-rate-limits

[^5]: OpenAI，"Error Codes"，OpenAI API Documentation，2026 年 6 月访问。 https://platform.openai.com/docs/guides/error-codes

[^6]: Marc Brooker，"Timeouts, Retries, and Backoff with Jitter"，Amazon Builders' Library，AWS，2023 年访问。 https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/

[^7]: LiteLLM / BerriAI，"Router - Load Balancing"，LiteLLM Documentation v1.86.0，2026 年 6 月访问。 https://docs.litellm.ai/docs/routing

[^8]: tianpan.co，"Idempotency Keys for Nondeterministic LLM Calls"，2026 年 5 月。 https://tianpan.co/blog/2026-05-16-retry-changed-answer-idempotency-keys-nondeterministic-llm
