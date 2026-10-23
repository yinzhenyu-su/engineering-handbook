---
title: "Token 与上下文窗口"
description: "Token 是 LLM 计费、限额和提示词的计量单位。如何在发送前用 Python 数清 token，以及非拉丁文字为何更贵。"
slug: 01-tokens-and-context-windows
part: 1
chapter: 1
difficulty: beginner
prerequisites:
  - part-0-start-here/00-what-an-ai-engineer-is
  - 00-the-model-landscape
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# Token 与上下文窗口

在读下一段之前，先跑一遍这段代码：

```python
# needs: pip install tiktoken
import tiktoken

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

english = "Hello, how are you today?"
japanese = "こんにちは、今日はお元気ですか？"

print(count_tokens(english))   # 7 个 token，25 个字符
print(count_tokens(japanese))  # token 多得多，字符少得多
```

两句问候意思一样。模型看到的不是字母，而是一串整数，而那句日文解析出的整数比英文多。你按整数付钱。所以同一句话换成另一种文字，在你连提示词的一个字都还没改的时候，就已经更贵，也吃掉更多固定预算。

这就是要关心 token（词元）的全部理由。价格、上下文上限、速率限制、延迟，全都按 token 算，不按单词或字符算。靠肉眼估字符长度来猜成本的工程师一定会错，有时错 4 倍，而且要等到 API 报错或账单寄来才发现。

## Token 是文本的一块碎片，不是单词

任何模型读你的提示词之前，先有一个分词器把字符串切成碎片，再把每块映射到固定词表里的一个整数 ID。模型在这些整数上运算，输出新的整数；第二道工序再把它们变回文本。

主流方法是 **Byte Pair Encoding**（BPE，字节对编码），截至 2026 年 6 月，OpenAI、Anthropic 和大多数厂商都在用。[^1] 训练期的细节你不用管，只需要记住三件事：

- **常见词就是一个 token。**“hello”“return”“function”各自映射到单个整数，因为它们在训练语料里出现得足够频繁。
- **生僻词会被切碎。**OpenAI 自己的例子：`"tiktoken is great!"` 变成六个 token，`['t', 'ik', 'token', ' is', ' great', '!']`。[^2] “tiktoken”这个词太生僻，于是被拆开。
- **切法是确定的，且跟模型绑定。**同一段文本在同一种编码下永远得到同一组 ID。但换个模型可能换一套词表，数量就变了。

有两个怪癖容易踩。前导空格会并到下一个词上，所以 `" is"` 是一个 token，不是两个。大小写也算数：句中的 `" red"` 和句首的 `"Red"` 是不同 token，ID 也不同。[^3]

就英文散文而言，OpenAI 给了一条经验值：大约 4 个字符一个 token，或者说一个 token 约 0.75 个单词。[^3] 粗估够用。一旦离开英文散文、碰到代码或粘进一个 URL，它就一点用没有。把它当合理性检查，永远别当护栏。

## 非拉丁文字更贵，而且差距很大

BPE 词表是用以英文为主的语料建起来的。常见的英文片段成了单独的 token。训练时见得少的文字则被切成很小的子词碎片甚至单个字节，同样的意思就要更多 token。

2025 年有一项研究用 tiktoken 的 `cl100k_base` 编码在 200 多种语言上测了这件事，指标是每 token 字符数（越高越省）。[^4]

![五种文字的每 token 字符数横向条形图，拉丁字母最高 2.61，藏文最低 0.49，英文经验值 4 标在图表之外](../../../ai-eng/part-1-how-models-behave/images/01-script-inflation.png)
*同一种编码下，拉丁字母一个 token 能装下的字符数是藏文的五倍多；非拉丁文字的用户花预算花得更快。*

实际影响很直接。该研究里最差的缅甸文，一句话塞进的 token 约为拉丁字母的 7 倍。[^4] 另一份 2023 年覆盖 22 种语言的分析说得更直白：许多非英语语言的使用者“被多收了钱，拿到的结果还更差”。[^5]

所以一段 1,000 token 的英文，换成印地语可能要 2,000 到 4,000 token。如果你的产品按你为英文定的字符预算，去服务日语、阿拉伯语或印地语用户，这些用户会撞上英文用户从没见过的截断和上下文报错。

规则是：**按目标文字实测 token。**英文那套 4 字符经验值对其他任何语言都偏低。把你从英文得出的估算乘上约 1.5 到 7 倍的文字系数，并且在定下上下文方案之前，用真实用户查询验证一遍。统一乘 2 并不安全，对缅甸文、藏文和达罗毗荼语系仍然不够。

## 发送之前先用 Python 数 token

每家厂商各有各的算法，彼此不能互换。

**OpenAI：tiktoken，本地、免费。**它跑在你机器上，不用发 API 请求，返回精确数字。用 `encoding_for_model(model)`，别写死编码名，因为不同模型系列用的编码不一样。GPT-4o 及更新模型用 `o200k_base`（20 万词表）；GPT-4 和 GPT-3.5 用 `cl100k_base`（10 万词表）。[^2] 本章开头那个 `count_tokens` 函数就是全部套路。

**Anthropic：count_tokens 端点。**tiktoken 没有模拟 Anthropic 的分词器，拿来算 Claude 会给出错误数字。用专门的 API，它免费，且限流与你主调用分开计（截至 2026 年 6 月）。[^6]

```python
# needs: pip install anthropic
import anthropic

def count_tokens_anthropic(messages, model="claude-sonnet-4-6"):
    client = anthropic.Anthropic()
    resp = client.messages.count_tokens(model=model, messages=messages)
    return resp.input_tokens

messages = [{"role": "user", "content": "Summarize the attached document."}]
# remaining = 200_000 - count_tokens_anthropic(messages)  # Haiku 4.5 = 20 万上下文
```

**Gemini：countTokens 方法。**Google 的 SDK 提供 `model.count_tokens()`，接受的 `contents` 结构跟你发给生成接口的一样。[^7]

有两项开销会从粗糙的计数里漏掉。聊天消息自带附加量：OpenAI 每条消息加 3 个 token，再加 3 个用于引导回复。[^2] 还有一项：**工具定义也算输入。**一个挂着 20 个工具的 agent，每次调用都可能多出几千 token 的 JSON schema。消息和工具都要数，否则你的护栏会在 API 拒掉请求的前一刻还显示通过。

上线之后，把每次响应里的 `usage.prompt_tokens` 记下来，跟发送前的估算对比。两者差超过 10%，说明你的估算器坏了，通常是因为用错了分词器，或者漏算了工具。

## 上下文窗口是一笔要花掉的预算

上下文窗口（context window）是一次调用中模型能处理的 token 上限，**输入加输出**。它是硬上限。越过去你拿到的是 `context_length_exceeded`，不是一句警告。

把它当成一个固定钱包，你发出去的一切和你想拿回来的一切都从这个钱包里出。

![一个标着上下文窗口的宽矩形，左半部分被分成若干叠放区块，依次为系统提示词、对话历史、检索到的文档、工具定义、用户轮次，右半部分留给输出](../../../ai-eng/part-1-how-models-behave/images/01-context-budget.png)
*提示词的每一部分都在抢同一个固定窗口；对话历史和工具定义花掉的额度，跟用户那个问题一样多。*

窗口在变大。截至 2026 年 6 月，GPT-5.5 和 Claude Sonnet 4.6 都提供 1,000,000 token 的输入窗口，Gemini 2.5 Flash 是 1,048,576。[^8][^9][^7] 但每个模型对输出另有单独上限，通常远低于整个窗口：Sonnet 4.6 即便输入窗口 1M，输出最多也只有 64k token。[^9]

三个习惯能让你待在预算之内。

- **给输出留位置。**先从事先留出一块固定的输出额度，再往里填输入。默认至少留 4,096 token；做代码生成或长文写作留 16k 到 32k。[^3] 省掉这一步，你就有可能在生成中途撞上上下文写满的错误。
- **盯住 agent 循环。**多轮 agent 每一轮都会重发完整历史，包括工具返回结果。一个 10 轮的任务，如果工具输出有 50k token，到第 10 轮就是 500k 输入 token，其中大部分是过时的。压缩策略对 agent 成本是承重结构，第 5 部分讲怎么把这笔预算花好。
- **换模型就重新数。**同一段文本，Anthropic 的 Claude Fable 5 分词器比 Opus 4.7 之前的 Claude 模型多产出约 30% 的 token。[^6] 从 GPT-4 迁到 GPT-4o 数量同样会变，非英文字符串尤其明显。任何模型切换之后，用新分词器把代表性提示词重新数一遍。

把护栏写成一个函数。它数输入、扣输出额度、拒绝发出装不下的请求。

```python
# needs: pip install tiktoken
import tiktoken

CONTEXT_LIMITS = {"gpt-5.5": 1_000_000, "gpt-5.4-mini": 400_000}
MAX_OUTPUT_RESERVE = 4_096

def check_fits(messages: list[dict], model: str = "gpt-5.4-mini") -> int:
    try:
        enc = tiktoken.encoding_for_model(model)
    except KeyError:
        enc = tiktoken.get_encoding("o200k_base")
    total = 0
    for m in messages:
        total += 3  # 每条消息的开销
        for v in m.values():
            total += len(enc.encode(str(v)))
    total += 3  # 回复引导
    budget = CONTEXT_LIMITS.get(model, 128_000) - MAX_OUTPUT_RESERVE
    if total > budget:
        raise ValueError(f"Prompt ({total} tokens) exceeds budget ({budget})")
    return total
```

还有一个坑值得单独提醒，因为它亏的是质量而不是钱。

> [!WARNING]
> **窗口更大，不代表模型会同等读到每一处。**模型表现出“中间遗失”效应：信息放在长上下文的最开头或最末尾时准确率最高，埋在中段就会下滑，即便是为长上下文专门做的模型也一样。[^10] 填满 1M 的窗口，并不保证模型真的关注了这 1M 个 token。把最相关的内容——检索到的片段和关键指令——放在开头或结尾附近。检索场景下把片段放在用户问题旁边，不要放在中间。

token 是计量单位。一旦你数得清，就能做这一部分其余内容所依赖的算术：把 token 数乘上每 token 的输入价和输出价，一次调用的成本就出来了。[成本、质量与延迟三角](./05-the-cost-quality-latency-triangle.md)把这套算术变成一个产品决策，它花的正是本章教你去度量的这个单位。

## 参考文献

[^1]: Sennrich, Rico; Haddow, Barry; Birch, Alexandra. "Neural Machine Translation of Rare Words with Subword Units." ACL 2016. arXiv:1508.07909. https://arxiv.org/abs/1508.07909
[^2]: Sanders, Ted (OpenAI). "How to count tokens with Tiktoken." OpenAI Cookbook, last updated Aug 2024. https://cookbook.openai.com/examples/how_to_count_tokens_with_tiktoken
[^3]: OpenAI. "What are tokens and how to count them?" OpenAI Help Center. Accessed June 2026. https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them
[^4]: Teklehaymanot, Hailay Kidu; Nejdl, Wolfgang. "Tokenization Disparities as Infrastructure Bias." arXiv:2510.12389, October 2025. https://arxiv.org/abs/2510.12389v1
[^5]: Ahia, Orevaoghene; et al. "Do All Languages Cost the Same? Tokenization in the Era of Commercial Language Models." EMNLP 2023. arXiv:2305.13707. https://arxiv.org/abs/2305.13707
[^6]: Anthropic. "Token counting." Anthropic API Documentation. Accessed June 2026. https://docs.anthropic.com/en/docs/build-with-claude/token-counting
[^7]: Google. "Gemini 2.5 Flash model card." Google AI for Developers. Last updated 2026-04-28. https://ai.google.dev/gemini-api/docs/models/gemini-2.5-flash
[^8]: OpenAI. "Models." OpenAI API Documentation. Accessed June 2026. https://platform.openai.com/docs/models
[^9]: Anthropic. "Models overview." Anthropic API Documentation. Accessed June 2026. https://docs.anthropic.com/en/about-claude/models/overview
[^10]: Liu, Nelson F.; et al. "Lost in the Middle: How Language Models Use Long Contexts." TACL 2024. arXiv:2307.03172. https://arxiv.org/abs/2307.03172
