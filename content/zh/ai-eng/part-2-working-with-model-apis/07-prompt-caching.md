---
title: "提示词缓存"
description: "API 原生 LLM 产品里最大的那根成本杠杆。一个 5 万 token 的 agent 跑 10 轮，改一处配置，账单从 $1.53 掉到 $0.47。"
slug: 07-prompt-caching
part: 2
chapter: 7
difficulty: beginner
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
  - 00-calling-models
tags: [llm, http-api, cache, cost]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 提示词缓存

一段 10 轮的对话，系统提示词 45,000 token，在 Claude Sonnet 4.6 上要花 $1.53。同一段对话，只改一行配置，花 $0.47。账单省掉 69%，每一个 agent 会话都是如此，产品跑多久就省多久。[^1] 大多数团队没去拨这个开关。有些人不知道它存在；另一些人以为它要 Redis 外加一个周末。都不用。在 OpenAI 上它就是字面意义的零代码。在 Anthropic 和 Gemini 上也就几行。

本章讲的是 API 原生 LLM 产品里最大的那根成本杠杆，以及为什么几乎没人用它。

## 你的账单为什么大半是同一批 token，一遍又一遍

每一次 LLM API 调用都是无状态的。厂商接过你的提示词，从第 0 个 token 开始跑模型，处理过的每个输入 token 都计费。同一个 45,000 token 的系统提示词发十次，你就为 450,000 个输入 token 付钱，哪怕其中 449,955 个和上一次调用逐字节相同。

在 GPU 内部，处理这些 token 会算出一叠注意力张量，叫做 KV 缓存（KV cache，即模型用来回看前面 token 的 key 和 value 矩阵）。没有提示词缓存时，响应一结束，厂商就把这些张量扔掉。下一个请求：从头再算。你为已经发生过的计算付了全价。

提示词缓存（prompt caching）就是厂商把这层 KV 缓存留着。当下一个请求以相同前缀开头，厂商跳过预填充（prefill），把命中缓存的 token 按基础输入价的一个零头计费：Anthropic 和 OpenAI 大约 10%，Gemini 是 25%。同一个模型，同样的输出，同样的答案。输入部分的账单最多省 90%，而且还更快——在长提示词上，首 token 时间（time-to-first-token）大部分花在预填充里。[^1][^2][^3]

## 算一遍账：10 轮，省下 $1.06

这是一个标准的 agent 形状：45,000 token 的系统提示词（工具定义、指令、检索来的文档），后面跟着一段每轮大约增长 1,000 token 的对话。在 Claude Sonnet 4.6 上按 2026 年 6 月的价格（每百万输入 $3.00，每百万缓存读 $0.30，每百万缓存写 $3.75）跑 10 轮：

```python
BASE  = 3.00  / 1_000_000   # $3.00/MTok 基础输入价
WRITE = 3.75  / 1_000_000   # 1.25 倍基础价，首次未命中时写一次
READ  = 0.30  / 1_000_000   # 0.10 倍基础价，之后每次缓存命中
OUT   = 15.00 / 1_000_000

STABLE = 45_000   # 系统提示词 + 工具 + RAG 上下文
TURN   = 1_000    # 每轮用户-助手交互新增的 token
TURNS  = 10

no_cache = sum(
    (STABLE + (t-1)*TURN + 500) * BASE + 200 * OUT
    for t in range(1, TURNS + 1)
)
with_cache = sum(
    (STABLE * WRITE + 500 * BASE + 200 * OUT) if t == 1
    else (STABLE * READ + (t-1)*TURN * BASE + 500 * BASE + 200 * OUT)
    for t in range(1, TURNS + 1)
)
print(f"No cache:   ${no_cache:.4f}")   # $1.5300
print(f"With cache: ${with_cache:.4f}") # $0.4703
print(f"Savings:    {(1 - with_cache/no_cache)*100:.1f}%")  # 69.3%
```

一个会话省一块钱，听着不多。并不是。一个 10,000 日活的产品，用户每天各跑一次这样的对话，就是每天 $10,600、每年 $3.9M 的输入 token——你本来就在付，而且完全不必付。再乘上你上线的每一个重提示词的功能。这就是为什么提示词缓存是那根杠杆。

省钱的形状和幅度一样重要。第一轮要付一笔小附加费（在 Anthropic 上，缓存写按 5 分钟档是 1.25 倍基础价，按 1 小时档是 2.0 倍）。[^1] 之后每一轮在稳定前缀上只付 10%。盈亏平衡点就是 TTL 窗口内的两次请求。如果你的流量足够突发，一个典型前缀在过期之前至少被复用一次，缓存就是稳赚的。

![两条累计成本曲线横跨十轮对话；上面那条珊瑚色线标注为 No cache，从第 1 轮的 $0 以陡直的斜率升到第 10 轮的 $1.53；下面那条靛蓝色线标注为 With cache，在第 1 轮略微跳升，之后以平缓得多的斜率上升，到第 10 轮停在 $0.47；两条线之间的差距被涂成阴影，并且随着每一轮不断变宽](../../../ai-eng/part-2-working-with-model-apis/images/07-cost-curve.png)
*十轮下来的累计成本。两条线之间的差距就是提示词缓存的价值，而且每多一轮就更宽一些。*

## 前缀设计只有一条规则：稳定的在前，易变的在后

缓存只在提示词开头跨请求逐字节相同时才生效。厂商对前缀做哈希；差一个字节，就完全未命中。所以这条规则在 Anthropic、OpenAI、Gemini 上都通用：把最稳定的内容放在提示词顶部，最易变的内容放在底部。

标准顺序，从上到下：

- **工具定义。** 一次部署内几乎从不变化。
- **系统指令。** 按部署变，不按请求变。
- **背景文档与 RAG 上下文。** 内容变了才变。
- **对话历史。** 每轮增长；较早的轮次会变成稳定部分。
- **当前用户消息。** 每个请求都是新的，永远在底部。

最常见的错误，是把一个每请求都变的值（时间戳、会话 ID、当天日期、用户名字）放在系统提示词顶部。这一个变量会让每个请求的哈希都失效，命中率变成 0%。你在响应的 usage 字段里立刻就能看出来：每次调用 `cache_creation_input_tokens` 都非零，而 `cache_read_input_tokens` 永远是零。如果这个变量挪不到末尾，就从提示词里彻底删掉，改走一次工具调用传进去。

举个例子：一个系统提示词以 `Today is {date}.` 开头，后面跟着 40,000 token 的稳定指令，结果什么都缓存不上。同样的提示词，把 `Today is {date}.` 挪到用户消息里，整整 40,000 token 就都缓存住了。到模型的信息一样，账单差一个数量级。

## 三家厂商，三种思路

机制是一样的，API 表面不是。每家都在开发者控制力和简单性之间押了不同的注，而这个注会体现在你的代码里。

### OpenAI：自动生效，零代码

OpenAI 对任何超过 1,024 token 的请求自动做提示词缓存。没有 `cache_control` 字段，也没有断点要设。厂商对你请求的一段前缀（通常是前 256 token）做哈希，把你路由到见过这段前缀的服务器上，然后按基础输入价的 10% 提供缓存的 KV 张量。你唯一要做的事，是把稳定内容放在前面。[^2][^4]

读响应里的 `usage.prompt_tokens_details.cached_tokens` 就能确认命中。对于许多请求并行发送同一段长前缀的高吞吐服务，`prompt_cache_key` 参数能帮路由器把同前缀的请求送到同一台机器；不给它的话，每个前缀每分钟超过约 15 个请求的突发就会溢出到别的机器上，直接未命中。[^2]

默认 TTL 是 5 到 10 分钟无活动，上限一小时。近期的旗舰模型（gpt-5.5、gpt-5.4、gpt-4.1 等）支持扩展的 24 小时模式，在显存满了的时候把 KV 张量卸载到 GPU 本地存储。对于复用间隔不到一分钟的聊天产品，默认值就够。对于每小时命中同一前缀的定时批处理任务，去申请扩展模式。[^2]

### Anthropic：显式断点，四个位置

Anthropic 把控制权直接交给你。你在提示词里最多标四个位置，用 `cache_control` 字段，厂商就把到每个标记块为止的前缀 KV 张量存下来。后续同前缀的请求直接从缓存读：

```python
import anthropic

client = anthropic.Anthropic()
SYSTEM_PROMPT = "... 45k tokens of stable documentation ..."

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},  # 5 分钟 TTL；要 1 小时用 "ttl": "1h"
    }],
    messages=[{"role": "user", "content": "What does section 4 say?"}],
)
u = response.usage
print(u.cache_read_input_tokens, u.cache_creation_input_tokens, u.input_tokens)
```

两个 TTL 选项：5 分钟（写按 1.25 倍基础价，读按 0.10 倍）或 1 小时（写 2.0 倍，读 0.10 倍）。用户可能喝完咖啡再回来，就选 1 小时；密集突发就选 5 分钟。[^1]

这份控制力附带两个值得了解的失效模式。第一是回看窗口：每个断点只能在自己前后 20 个消息块之内找到缓存条目。一段增长超过 20 轮的对话会悄悄开始未命中，除非你在稳定系统提示词的末尾也放一个断点——那个断点总会被走到。第二，缓存前缀里的任何东西都必须逐字节一致：换一张图片、切换 `tool_choice`、改动工具定义，都会让缓存失效。[^1]

一个有用的预热技巧：应用启动时发一个 `max_tokens=1` 的请求，在用户流量到来之前把缓存填上（Messages API 至少要求 1 个输出 token）。API 处理完前缀、写好缓存，为这个特权收你一个输出 token 的钱。[^1]

### Gemini：显式的命名缓存资源

Gemini 走了第三条路。你用 `client.caches.create()` 在服务端建一个缓存资源，拿回一个名字，然后在后续的 `generate_content` 调用里引用这个名字：

```python
from google import genai
from google.genai import types

client = genai.Client()
cache = client.caches.create(
    model="models/gemini-2.5-pro",
    config=types.CreateCachedContentConfig(
        display_name="doc_cache",
        contents=[types.Content(role="user", parts=[types.Part(text=DOCUMENT)])],
        ttl="3600s",  # 默认 1 小时；之后可以更新
    ),
)

response = client.models.generate_content(
    model="models/gemini-2.5-pro",
    contents="Summarize section 4.",
    config=types.GenerateContentConfig(cached_content=cache.name),
)
```

这个形状适合另一种负载：一份大的稳定语料（一段 10 分钟的视频、一本 500 页的 PDF、一整个代码库），在几小时或几天里被反复查询。TTL 可以配到任意长度；Anthropic 的上限是一小时。代价是读价之外还有一笔按 token 按小时的存储费，而且 Gemini 的缓存读按基础输入价的 25% 收，不是 Anthropic 和 OpenAI 的 10%。在 Gemini 2.5 Pro 上，每小时每百万 token $4.50，把 100k token 缓存 24 小时的存储费是 $10.80；而每读一百万缓存输入 token 省 $0.94，那 24 小时里你得对这份缓存查大约 115 次才回本。[^3][^5] 低于这个查询频率，开缓存比不开更贵。Gemini 2.5 及更新版本还会自动启用隐式缓存（不需要建资源），但不保证命中。

## 什么时候该默认用哪家

决策取决于你已经在哪、以及在缓存什么：

- **你在 OpenAI 上。** 已经做完了。确认稳定内容在提示词顶部，在日志里看 `cached_tokens`，上线。
- **你在 Anthropic 上，有长系统提示词或多轮 agent。** 在稳定系统提示词末尾放一个显式断点，默认用 5 分钟 TTL；如果用户经常一停就是五分钟以上，就换成 1 小时。
- **你在 Gemini 上缓存一份大的稳定语料，批量查询。** 用显式的命名缓存，并且先把回本账算清楚。
- **你的提示词不到 1,024 token，或者每个请求的前缀都不同。** 别缓存。低于下限时厂商正常处理；没有共享前缀，就没有可共享的东西。

团队跳过缓存最常见的原因，是以为它很难。并不难。在 OpenAI 上，你现在正为已经在跑的缓存付着全价。在 Anthropic 上，一个内容块上的一个字段就是全部集成工作。难的部分是前缀纪律，而那只是对“你的变量都放在哪”做一次性的检查。

要看更大的成本图景（成本-质量-延迟三角，以及缓存之外的其他杠杆），[成本、质量、延迟三角](../part-1-how-models-behave/05-the-cost-quality-latency-triangle.md)给出了本章所处的框架。当单靠缓存不够时，[成本工程](../../../ai-eng/part-10-deployment-and-cost/03-cost-engineering.md)会把它和批处理、路由、量化、蒸馏叠在一起用。

## 参考文献

[^1]: Anthropic, "Prompt caching", Anthropic API Docs, https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching (fetched June 12 2026)
[^2]: OpenAI, "Prompt caching", OpenAI API Docs, https://platform.openai.com/docs/guides/prompt-caching (fetched June 12 2026)
[^3]: Google, "Context caching", Gemini API Docs, https://ai.google.dev/gemini-api/docs/caching (fetched June 12 2026)
[^4]: OpenAI, "Pricing", OpenAI API Docs, https://platform.openai.com/docs/pricing (fetched June 12 2026)
[^5]: Google, "Gemini Developer API pricing", https://ai.google.dev/gemini-api/docs/pricing (fetched June 12 2026)
