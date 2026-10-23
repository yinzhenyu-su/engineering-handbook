---
title: "推理模型"
description: "什么是测试时计算，各家厂商的力度与预算旋钮，为什么你要为隐形的思考 token 付费，以及推理在什么时候值这个价。"
slug: 04-reasoning-models
part: 1
chapter: 4
difficulty: beginner
prerequisites:
  - 00-the-model-landscape
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 推理模型

以 high 力度调用一次 Claude Opus 4.8，可能只返回 200 个词可见文本，却按 5,000 个你根本看不见的 token 向你收费。按 2026 年 6 月每百万输出 token 25 美元算，这一次调用大约 0.13 美元。同样 200 个词，标准模型大约 0.008 美元。屏幕上答案一样，账单是 16 倍。[^1]

这个差距就是本章存在的全部理由。推理模型先想再答，思考过程以 token 的形式发生，而每一个这样的 token 你都按完整的输出价格付费，尽管它们默认对你不可见。这一点弄错，流量没变、月账单涨 3 到 5 倍；弄对了，你手上就多了一件能干多步推理、而标准模型会搞砸的工具。

## “思考”到底指什么

把营销话术剥掉，机制本身很朴素。在写出可见回复之前，推理模型会先生成一长串内部 token，也就是它的思维链（chain of thought）。这些 token 进入模型自己的上下文，引导它接下来写什么。然后它们被丢掉。[^1]

研究者把这件事叫**测试时计算（test-time compute）**：模型不是在输入上过一遍就完事，而是在推理时额外花算力去规划、检查备选方案、在走错路后回头，然后才给出答案。标准模型只有一次机会，推理模型多了一块草稿纸。

草稿纸上的 token 就是代价所在。它们按可见输出的同一单价计费，而多数厂商不给你看。截至 2026 年 6 月，在所有主流厂商那里，这些思考 token 都有三个共同点：

- **计费。** 单价与可见答案相同。
- **不可见。** 你最多拿到一份摘要，永远拿不到原始思维链。
- **轮次之间丢弃。** 模型读不到自己上一条消息里的推理，所以每一轮都从头再想。[^1]

响应对象里能查到真实数量。OpenAI 的 Responses API 是 `output_tokens_details.reasoning_tokens`；Anthropic 是 `output_tokens_details.thinking_tokens`；Google 的 Gemini API 是 `usage_metadata.thoughts_token_count`，与可见的 `candidates_token_count` 分开上报。[^1][^2][^3] 去查它。你的账单取决于它。

![一次 API 请求分成两条并行的 token 流：一条标明为隐形但计费的思考 token 的浅色大流，注入一条可见输出 token 的深色小流，两股都汇入一个响应对象](../../../ai-eng/part-1-how-models-behave/images/04-two-token-streams.png)
*一次调用产生两条计费流。你读小的那条，付两条的钱。*

## 旋钮：每家厂商各有一个

与其说你在开关思考，不如说你在拧“模型要想多用力”的旋钮。每家厂商暴露的控制面不一样，名字也对不上，所以三种都列在这里供你查。

**OpenAI** 在 Responses API 上用 `reasoning.effort`。接受 `none`、`minimal`、`low`、`medium`、`high`、`xhigh`。GPT-5.5 默认 `medium`。[^1]

```python
from openai import OpenAI

client = OpenAI()  # 从环境变量读取 OPENAI_API_KEY

response = client.responses.create(
    model="gpt-5.5",
    reasoning={"effort": "low"},  # none | minimal | low | medium | high | xhigh
    input=[{"role": "user", "content": "Write a bash script to transpose a matrix."}],
)

print(response.output_text)
# 真正决定账单的数字，不是可见 token 数：
print("Reasoning tokens used:", response.usage.output_tokens_details.reasoning_tokens)
```

**Anthropic** 现在推荐自适应思考：模型按每个请求自己决定想多少，你用 `effort` 去塑造这个量。旧的手动 `budget_tokens` 在 Sonnet 4.6 及更早版本上还能用，但在 Opus 4.6+ 与 Sonnet 4.6+ 上已废弃。[^2]

```python
import anthropic

client = anthropic.Anthropic()  # 从环境变量读取 ANTHROPIC_API_KEY

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # low | medium | high | max | xhigh
    messages=[{"role": "user", "content": "Solve this optimization problem step by step."}],
)

# 即使你只看到推理的摘要，这部分照样计费：
print("Billed thinking tokens:", response.usage.output_tokens_details.thinking_tokens)
```

**Google Gemini** 按代际分开。2.5 系列用 `thinkingBudget`，一个整数 token 上限（Flash 上 0 到 24,576，Pro 上 128 到 32,768）。3 系列用 `thinkingLevel`，枚举值 `minimal`、`low`、`medium`、`high`。Gemini 3.5 Flash 默认 `medium`；3.1 Pro 默认 `high`，且不能低于 `low`。[^3]

```python
from google import genai
from google.genai import types

client = genai.Client()  # 从环境变量读取 GOOGLE_API_KEY

response = client.models.generate_content(
    model="gemini-3.5-flash",
    contents="Explain Occam's Razor with an everyday example.",
    config=types.GenerateContentConfig(
        thinking_config=types.ThinkingConfig(thinking_level="low"),
    ),
)

print("Thinking tokens:", response.usage_metadata.thoughts_token_count)
print("Output tokens:", response.usage_metadata.candidates_token_count)
```

有两个怪癖容易咬人。第一，有些模型根本不让你关掉思考：Gemini 2.5 Pro 拒绝 `thinkingBudget=0`，Anthropic 的 Fable 5 则始终开着自适应思考。[^2][^3] 第二，延迟旋钮和成本旋钮是同一个旋钮。力度调高意味着更多思考 token，也就同时意味着更贵的账单和更慢的响应。没有任何设置能让你免费多换一点推理。

## 预算陷阱：付了钱却什么也没拿到

思考 token 和可见答案共用同一份输出预算。`max_output_tokens`（OpenAI）或 `max_tokens`（Anthropic）设得太低，模型可能把整份预算都烧在思考上，然后带着 `status: "incomplete"` 和零可见文本返回。输入和它消耗的每个思考 token，你照样付钱。[^1]

OpenAI 自己的文档把这个失败情形写得很直白：模型可能在“任何可见输出 token 产生之前”就撞上上限，“这意味着你可能为输入和推理 token 付了费，却没收到可见回复”。[^1] 他们给的解法很硬：为推理至少留出 25,000 token 的余量。[^1] Anthropic 另外提醒，超过 32k token 的思考预算应该走它们的批处理端点，因为实时请求想那么久会撞上 HTTP 超时。[^2]

所以在上线一个推理调用之前，两条规矩：

- **给它留地方。** token 上限要能盖住思考加答案，而不只是答案。
- **查状态。** 每个响应都读一下 `status`（OpenAI）或 `stop_reason`（Anthropic）。出现 `incomplete` 或 `max_tokens`，说明你付了钱却没拿到有用的东西。

## 推理什么时候值这个价，什么时候不值

默认用标准模型。推理模型是一次要用测量结果来正当化的升级，而不是一个开着就不管的设置。OpenAI 自己的指导也是这个说法：从标准路径起步，只有当“质量与智能比延迟更重要”时才上 high 力度。[^1]

当任务可以拆成多步、且中间结果要喂给后面的步骤时，才该上推理模型：

- **多步数学与算法代码。** 前面错一步就会污染后面全部的问题。
- **智能体工作流（agentic workflow）。** 不确定性下的工具编排与决策树，这是生产环境最主要的用例。[^1]
- **错误代价高的任务。** 安全审查、不带测试就发布的代码，任何“第一版答错、后面发现很贵”的事情。

下列情况关掉它，或者把力度压到最低：

- **延迟要求紧。** 语音、联想输入、流式聊天这类亚秒级 SLA。OpenAI 建议语音快路径用 `effort: "none"`。[^1]
- **任务是单步的。** 分类、JSON 转 CSV、短抽取，标准模型一遍就能做对的查询。
- **吞吐量比深度更重要。** 可并行的批量活儿，交给便宜的标准模型走批处理端点。

最贵的错误是在不需要的任务上跑高力度。一次情感分类调用，在 Claude Opus 4.8 上用 `high` 力度，回答一个“positive”也可能烧掉 1,000 到 3,000 个思考 token。[^2] 它推理是因为它被训练成要推理，不是因为任务需要。盯住你的可观测性数据：如果某类任务稳定花 500 个以上思考 token 才产出一个词，就降它的力度，或者把它路由到更便宜的模型。

| | 延迟严格（1 秒以内） | 延迟宽松（5 秒以上） |
|---|---|---|
| **简单任务** | 标准模型 | 标准模型（推理纯属浪费钱） |
| **复杂任务** | 只能用低力度，或者蒸馏 | 推理模型，用评测结果调力度 |

真要提高力度时，从旋钮底部往上爬。Anthropic 的建议是“从最小思考预算起步，逐步增加”，并指出对多数任务而言，超过 32k 思考 token 之后的准确率收益已经很边际。[^2] 在你自己的任务上跑评测证明 `high` 胜过 `medium` 之前，别为它付钱。成本、质量、延迟三者的完整框架在[成本—质量—延迟三角](./05-the-cost-quality-latency-triangle.md)那一章；力度旋钮只是它的杠杆之一。

## 一个警告：别急着套用你的旧提示词

你学到的所有“从标准模型那里套出更好答案”的技巧，在这里可能起反作用。推理模型是被训练成自己生成思维链的，所以替它规定步骤——写死“think step by step”这类指令，或给少样本范例——反而可能打断它本会找到的更好路径。有一项研究测到，在“深思反而有害”的任务上给 o1-preview 套思维链提示，准确率最多下降 36.3 个百分点。[^4] Anthropic 自己的指导是：“像‘think thoroughly’这样的提示，往往比人手写的分步计划产生更好的推理。”[^5]

不要把给标准模型的少样本提示词直接粘进推理模型，然后假定它们有用。思维链、ReAct 与自洽性那一章完整讲了这种反转，也讲了哪种技术适配哪一类模型。眼下你只要给推理模型目标、输出格式和约束，然后别挡它的路。

## 参考文献

[^1]: OpenAI，“Reasoning models”，OpenAI 开发者文档（2026 年 6 月访问），https://platform.openai.com/docs/guides/reasoning
[^2]: Anthropic，“Building with extended thinking”，Anthropic 开发者文档（2026 年 6 月访问），https://docs.anthropic.com/en/docs/about-claude/models/extended-thinking-models
[^3]: Google，“Gemini thinking”，Google AI for Developers（最后更新于 2026 年 6 月 4 日），https://ai.google.dev/gemini-api/docs/thinking
[^4]: Ryan Liu 等，“Mind Your Step (by Step): Chain-of-Thought can Reduce Performance on Tasks where Thinking Makes Humans Worse”，arXiv:2410.21333，2024 年 10 月 27 日提交，2025 年 6 月 13 日修订，https://arxiv.org/abs/2410.21333
[^5]: Anthropic，“Prompting best practices”，Anthropic 开发者文档（2026 年 6 月访问），https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices
