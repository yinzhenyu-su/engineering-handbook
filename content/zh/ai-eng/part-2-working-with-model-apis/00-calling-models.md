---
title: "调用模型"
description: "一次 chat completion 的 POST 请求背后发生了什么：消息、角色、系统提示词，以及你必须并排看清的三家 provider 请求形态。"
slug: 00-calling-models
part: 2
chapter: 0
difficulty: beginner
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-1-how-models-behave/02-sampling-and-temperature
tags: [llm, http-api]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 调用模型

所谓“chat completion”，就是一次 HTTP 请求。你发一个消息列表，拿回一条消息。所有主流 provider 最后都收敛到这层抽象上，而“概念上懂”和“真正在线上跑通”之间的距离，正是大多数工程师第一天耗掉的地方。

下面是最小可用的调用，对着你实际会用的三家 provider 各写一遍：

```python
# OpenAI
from openai import OpenAI
client = OpenAI()
r = client.chat.completions.create(
    model="gpt-5.4",
    messages=[
        {"role": "system", "content": "You are a concise assistant."},
        {"role": "user", "content": "What is the capital of France?"},
    ],
    max_tokens=50,
    temperature=0.0,
)
print(r.choices[0].message.content)
```

```python
# Anthropic
import anthropic
client = anthropic.Anthropic()
m = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=50,
    system="You are a concise assistant.",
    messages=[{"role": "user", "content": "What is the capital of France?"}],
    temperature=0.0,
)
print(m.content[0].text)
```

```python
# Gemini
from google import genai
from google.genai import types
client = genai.Client()
r = client.models.generate_content(
    model="gemini-3.5-flash",
    contents="What is the capital of France?",
    config=types.GenerateContentConfig(
        system_instruction="You are a concise assistant.",
        max_output_tokens=50,
        temperature=0.0,
    ),
)
print(r.text)
```

三次调用，同一个答案。但眯眼细看，差别并不是表面功夫。系统提示词住在三个不同位置。`max_tokens` 对其中一家是必填，对其余两家可选。文本出现在响应对象里三条不同路径上。Anthropic 连“assistant”这个词的用法都和别人不一样。

这就是本章的内容：角色、系统提示词、请求生命周期，以及你把同一个调用接到第二家 provider 时会踩到的那几个结构性差异。

## 消息是一串带类型的对话轮次

在网络传输层面，一次对话就是一个有序列表。每个元素有一个 **role**（谁在说话）和一个 **content**（说了什么）。模型每次调用都把整个列表读一遍，生成下一条 assistant 轮次，然后停下。它不记得上一次的任何东西。想做多轮对话，你得把 assistant 的回复追加回列表，下一次请求时把整个列表再发一遍。

这句话值得说两遍，因为它决定了后面所有事情：**API 是无状态的，对话历史由你负责，不是 provider 负责。** 每一轮，完整历史都会重新进入上下文窗口，从头重新分词。输入 token 成本随对话长度线性增长。（OpenAI 较新的 Responses API 提供了可选的 `previous_response_id` 做服务端串联，但那些历史 token 仍然按输入计费。[^1]）

生产代码里 95% 的场景只用到三种角色：

- **`system`**（在 OpenAI 的推理模型上叫 `developer`）：开发者给的指令。人设、规则、输出契约。信任级别最高。在对话开始时设一次，之后每一轮复用。
- **`user`**：一切来自外部的输入，主要是最终用户的文字。信任级别较低。提示词注入就住在这里，因为 `user` 内容里的任何东西都是别人写的，不是你写的。
- **`assistant`**：模型上一次说的话。你把之前的 assistant 轮次回填进 `messages`，模型才能看到自己先前的回答并保持连贯。

还有两种角色留给工具调用（OpenAI 的 `tool`，以及 Anthropic 的 `tool_use` 内容块），但它们属于[函数与工具调用](./03-function-and-tool-calling.md)那一章。

## 系统提示词放在三个不同地方

这是所有做代码迁移的人都会踩的跨 provider 陷阱。同一个东西——开发者给出的权威指令——在三家 provider 上住在不同字段里：

![并排的三份请求体，展示同一段提示词在 OpenAI Chat Completions、Anthropic Messages 和 Gemini generateContent 中的位置，系统提示词在每一份里高亮的位置都不一样](../../../ai-eng/part-2-working-with-model-apis/images/00-system-prompt-placement.png)
*同一条指令，三个位置：OpenAI 是 `messages` 里的 `system` 角色，Anthropic 是顶层 `system` 字段，Gemini 是顶层 `systemInstruction` 字段。*

这个分叉很要紧，因为它决定了你可移植的代码怎么写。如果你把系统提示词当成普通消息，把同一份 `messages` 列表往三家身上一套，Anthropic 会返回 400（“system not allowed in messages”），Gemini 直接不认这个角色名，然后你花一小时 debug。

防御性写法很简单：在代码里把系统提示词单独放一个变量，按 provider 注入到对应字段。永远不要把系统提示词烤进那个对话列表里存着。

还有一个 OpenAI 特有的坑。在 o1 和更新的推理模型上，规范角色是 `developer`，不是 `system`。眼下两者还能互换使用，但模型规范写得很明确：高信任指令要靠 developer 消息送达推理模型，用 `developer` 在 agent 类任务上指令遵循更好。[^2] o 系列和 gpt-5.x 推理模型默认用 `developer`；其余场景用 `system`。

Anthropic 在 `messages` 里也把模型输出叫 `assistant`。Gemini 在 `contents[].role` 里叫 `model`。这是仅次于系统提示词的第二大迁移 bug。给 Gemini 回填之前的 assistant 轮次时，角色字符串是 `"model"`，不是 `"assistant"`。

## `create()` 到拿到响应之间发生了什么

这个生命周期比大多数人想的短。SDK 把你的参数序列化成 JSON，POST 到 provider 的 HTTPS 端点，然后等。在 provider 那一侧，输入字符串被分词，模型一个 token 一个 token 地解码，直到碰到停止条件（停止序列、回合结束 token，或你设的 `max_tokens` 上限），结果再以 JSON 返回。你拿到一个响应对象，里面有生成的文本、token 用量明细，以及一个 `finish_reason` 告诉你为什么停止生成。

最后这个字段是新手最容易忽略、也最不该忽略的。它告诉你模型是说完了，还是被截断了：

- `stop`（OpenAI，Gemini 是 `STOP`）或 `end_turn`（Anthropic）：模型自然结束。这是你想要的。
- `length`（OpenAI，Gemini 是 `MAX_TOKENS`）或 `max_tokens`（Anthropic）：模型在半句话处撞到输出上限。回复被截断。要么调高 `max_tokens`，要么等着收一段残缺的 JSON。
- `tool_calls` / `tool_use`：模型想调工具。处理方式见[函数与工具调用](./03-function-and-tool-calling.md)。
- `content_filter`（OpenAI）、`SAFETY` / `RECITATION` / `PROHIBITED_CONTENT`（Gemini）、`stop_sequence`（Anthropic）：provider 出手干预了，或者你撞上了停止序列。生产环境里值得打一条日志。

文本本身在响应对象里也有三条不同路径，烦人但躲不开：

```python
# OpenAI Chat Completions
text = response.choices[0].message.content

# Anthropic Messages
text = response.content[0].text

# Gemini
text = response.text  # SDK 便捷属性；原始路径是 candidates[0].content.parts[0].text
```

Anthropic 的 `content` 即使是纯文本回复也永远是带类型块的列表，因为同一个字段在更复杂的响应里还要装 tool-use 块和图片块。不要假设 `content` 是字符串。

每次响应都会带回 token 用量，每次调用你都该把它记进日志。成本监控不是另一套可观测性问题，它就在响应里：

```python
# OpenAI: response.usage.prompt_tokens, response.usage.completion_tokens
# Anthropic: response.usage.input_tokens, response.usage.output_tokens
# Gemini: response.usage_metadata.prompt_token_count, .candidates_token_count
```

把这些数字乘上你的单 token 价格（截至 2026 年 6 月，gpt-5.4 是每百万输入 token $2.50、每百万输出 token $15.00[^3]；claude-sonnet-4-6 是 $3.00 / $15.00[^4]；gemini-3.5-flash 是 $1.50 / $9.00[^5]），你就能实时拿到每次调用的成本，不需要账单看板。

## 换 provider 时会咬人的参数

大多数生成参数在各家 provider 上含义一致，但有三个例外，上线前值得先搞清楚。

**Anthropic 上 `max_tokens` 是必填。** 没有默认值。不带它的请求会返回 400，报 `max_tokens: field required`。OpenAI 和 Gemini 都允许省略，一直生成到模型自然停止或撞到上下文窗口边界。[^6] 如果你的代码就是围绕“不传 `max_tokens`”写的，那么每一个 Anthropic 调用都会失败，直到你补上它。

**温度的取值范围不一样。** OpenAI 接受 `0.0` 到 `2.0`。Gemini 接受 `0.0` 到 `2.0`。Anthropic 上限是 `1.0`。[^6] 给 Anthropic 传 `temperature=1.5` 会得到 422。如果你用一个配置项同时驱动多家 provider，为了可移植性就把它限制在 `[0.0, 1.0]`。

**`top_k` 只有 Anthropic 和 Gemini 有。** OpenAI 不暴露它。你在一家上调好的 `top_k`，在另一家根本不存在这个旋钮。

这些参数在概念层面对 token 分布到底做了什么，第 1 部分的[采样与温度](../part-1-how-models-behave/02-sampling-and-temperature.md)已经讲过。落到调用代码上，规则是：只要你希望同样输入产出同样输出（分类、结构化抽取、工具调用），就默认 `temperature=0.0`；创意生成用 `0.7` 到 `1.0`。单靠温度拿不到你想要的效果之前，别去动 `top_p` 或 `top_k`。

## 说说 OpenAI 的两套 API

OpenAI 出了两个干同一件事的端点。**Chat Completions**（`/v1/chat/completions`）是 2023 年以来所有教程都在用的那个。**Responses**（`/v1/responses`）更新，截至 2026 年 6 月，OpenAI 推荐所有新项目用它。[^1] 思路一样，形状略有不同：`messages` 变成 `input`，系统消息变成顶层 `instructions` 字段，响应返回 `output[]` 而不是 `choices[]`，还可以用 `previous_response_id` 和 `store: true` 拿到可选的服务端状态。

学习阶段用 Chat Completions 完全够，本章的代码片段也用它，因为它在开源生态里读起来一模一样（LiteLLM、OpenAI Python SDK 的 `base_url` 覆盖、各种框架都认）。OpenAI 上的新生产代码默认用 Responses，除非你的框架还没跟上。第三个更老的选择 Assistants API 将在 2026 年 8 月下线。[^1]

一个常见误读：传 `previous_response_id` 并不能让之前的 token 不计费。每次调用它们仍然按输入计费。[^1] Responses API 省的是状态管理，不是成本。要省钱用[提示词缓存](./07-prompt-caching.md)，那才是对缓存输入 token 的真实折扣。

## 一个真正值回成本的跨 provider 抽象

你迟早会想在不重写调用代码的前提下换 provider。有两种做法可行：

**LiteLLM。** 一个包装层，对 100 多家 provider（包括 Anthropic 和 Gemini）暴露 OpenAI 兼容的 `completion()`。你只改 `model` 字符串（`"anthropic/claude-sonnet-4-6"`、`"gemini/gemini-3.5-flash"`），字段映射由 LiteLLM 处理，包括系统提示词放在哪。2026 年跨 provider 的生产代码默认走这条路。

**Gemini 的 OpenAI 兼容端点。** Google 在 `https://generativelanguage.googleapis.com/v1beta/openai/` 提供了一个 OpenAI 形状的端点。[^7] 用 `base_url=` 把它指过去、`api_key=` 填你的 Gemini key，OpenAI 格式的请求就能打到 Gemini 模型上。加不了依赖时很好用。

这两种做法都没有消除底层差异，只是把它们糊住了。真出问题时（工具 schema 被拒、参数被忽略、流式分块形状不一样），你还是得回到 provider 原生层面 debug。架构层面的讨论，[模型路由器与网关](../../hld/part-8-case-studies/37-model-router-gateway.md)讲了“用一个 API 挡住多家 provider”的白板视角。

比任何包装层都更能让代码长久干净的做法是：在代码库里写一个薄的 `call_model(messages, model, **kwargs)` 函数，在里面按 provider 分支，让应用层其余代码永远看不到任何厂商专有字段。你换 provider 的那天，就是改一个文件的那天。

## 参考文献

[^1]: OpenAI，“Migrate to the Responses API”，OpenAI Developer Docs，https://platform.openai.com/docs/guides/migrate-to-responses（访问于 2026 年 6 月）。
[^2]: OpenAI，“API Reference: Chat Completions”，https://platform.openai.com/api/reference/resources/chat（访问于 2026 年 6 月）。
[^3]: OpenAI，“Pricing”，https://platform.openai.com/docs/pricing（访问于 2026 年 6 月）。
[^4]: Anthropic，“API Pricing”，https://claude.com/pricing（访问于 2026 年 6 月）。
[^5]: Google，“Gemini Developer API pricing”，https://ai.google.dev/gemini-api/docs/pricing（最后更新于 2026-06-09）。
[^6]: Anthropic，“Messages API reference”，https://docs.anthropic.com/en/api/messages；Google，“generateContent method reference”，https://ai.google.dev/api/generate-content（均访问于 2026 年 6 月）。
[^7]: Google，“OpenAI compatibility”，https://ai.google.dev/gemini-api/docs/openai（访问于 2026 年 6 月）。
