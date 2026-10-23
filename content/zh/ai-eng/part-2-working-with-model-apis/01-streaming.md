---
title: "流式输出"
description: "逐 token 的流式输出如何在感知延迟上取胜，它为什么会破坏输出校验，以及同时拿到两者的四种做法。"
slug: 01-streaming
part: 2
chapter: 1
difficulty: beginner
prerequisites:
  - 00-calling-models
tags: [llm, http-api]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 流式输出

一次非流式请求打到 GPT-5.5，500 token 的回答大约六秒才回来。这六秒里用户只能盯着转圈。同一次调用改成流式，第一批词一秒左右就上屏，剩下的随模型写随滚出来。[^1]应用从“活着”变成“坏了”，差别就在这一处改动，而模型侧你一分钱不多花。

但它在别处要你付代价，而这正是多数教程跳过的一段。token 一开始往用户那边流，你就失去了在他们看到之前先看一眼完整回答的机会。schema 校验、内容审核（moderation）、语义护栏，凡是需要拿到完整答案才能干活的东西，全都只能在用户已经读过之后才跑。整章的张力就在这里。

## 线上实际传的是什么

传输层用的是 Server-Sent Events（SSE），一个 2000 年代浏览器就会说的纯 HTTP 协议。[^2]你开一条 HTTP 连接，服务端把它一直挂着，模型每产出一个 token 就往里写一行 `data: {...}\n\n`。客户端边到边读。生成结束，连接关闭。

你几乎不会直接碰原始字节。厂商 SDK 把解析藏起来，交给你一个产出类型化事件的 Python 迭代器。下面是 OpenAI 的写法：

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def stream_reply(prompt: str) -> str:
    full = ""
    stream = client.chat.completions.create(
        model="gpt-5.5",
        messages=[{"role": "user", "content": prompt}],
        stream=True,
    )
    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content is not None:
            print(delta.content, end="", flush=True)  # 在这里渲染
            full += delta.content
    return full
```

这个循环有三件事值得知道。第一个 chunk 带角色（`"assistant"`），但没有文本，所以 `delta.content` 是 `None`，跳过它。中间的 chunk 各带一小段字符串，通常几个 token。最后一个 chunk 是空的，带 `finish_reason`。OpenAI SDK 在 `data: [DONE]` 这个哨兵值处结束迭代。[^3]

Anthropic 的报文更丰富。每个事件单独一行，带一个类型名，于是你按事件类型分支：

```python
import os, anthropic

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def stream_claude(prompt: str) -> str:
    full = ""
    with client.messages.stream(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        for event in stream:
            if event.type == "content_block_delta" and event.delta.type == "text_delta":
                full += event.delta.text
            elif event.type == "message_delta" and event.delta.stop_reason == "refusal":
                raise RuntimeError("streaming refusal; reset conversation context")
    return full
```

上面那句 `message_delta` 检查不是装饰。Anthropic 是为 2026 年的一种失败模式加的，我马上会回来说。[^4]

## 流式为什么感觉快这么多

用户真正感受到的指标是首 token 延迟（time-to-first-token，TTFT），即从发出请求到看见任何输出的间隔。它和总生成时间不是一个数，却主导了感知延迟（perceived latency）。[^1]

![两条水平时间条，分别标注流式与非流式。流式那条先有一小段空白，然后文字从左往右填满。非流式那条是一长段空白，最后整段回答一次性落地。流式条上有一个珊瑚色标记，标出第一个字符出现的位置。](../../../ai-eng/part-2-working-with-model-apis/images/01-ttft-vs-full-response.png)
*流式付出同样的总生成成本，但用户在 TTFT 就看到输出，而不是等到最后。*

这些数字随模型和厂商浮动。截至 2026 年 4 月，Claude Sonnet 4.6 的 TTFT P50 约 0.74 秒；GPT-5.5 标准版 1.12 秒；Groq 的 LPU 这类专用推理硬件把 Llama 4 压到 0.18 秒。[^1]推理模式是另一个星球。Claude Opus 4.7 在扩展思考（extended thinking）下 P50 TTFT 是 28 秒。GPT-5.5 Pro 在高 `reasoning_effort` 下到 67 秒。[^1]TTFT 一旦越过 5 秒左右，流式对感知响应速度就不再有帮助，因为用户早就放弃等那个转圈了。推理模式要给加载态，不要给流式输出。

## 代价：流式让校验失效

流式从这儿起就不再是白赚的。JSON schema 校验、内容审核、语义护栏，还有“这个回答有没有检索到的文档撑着”这类分类器，都需要同一件东西：完整的输出。一个流式 JSON 对象的每一个中间状态，按定义都是语法非法的。[^5]你没法对 `{"name": "Al` 跑 `json.loads`。你也没法问一个审核分类器半句话有没有害，因为半句话还不算一句话。

OpenAI 在文档里说得很直白：流式“让审核补全内容变得更困难，因为部分补全更难评估”，审核分数“要等完整生成输出可用之后才到，不会随部分输出的 delta 一起给出”。[^3]

于是你面前是个岔路口。想让用户在生成过程中就看到回答，就没法在他们看到之前校验。想在用户看到之前校验，就没法边生成边展示。自己选。

## 四条出路

**先流式，事后校验。** token 到了就渲染到界面，但绝不据此行动。把完整响应缓存下来，流关闭后解析一次，然后才交给下游的任何东西。校验失败就把可见文字换成错误提示，或者一句“我再试一次”的重试。这是聊天场景的正确默认：消费者是人，内容本身可读，半截渲染闪一下也能挽回。

```python
import json, os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def stream_then_validate(prompt: str, required: list[str]) -> dict:
    buffer = ""
    stream = client.chat.completions.create(
        model="gpt-5.5",
        messages=[
            {"role": "system", "content": "Respond with valid JSON only."},
            {"role": "user", "content": prompt},
        ],
        response_format={"type": "json_object"},
        stream=True,
    )
    for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content is not None:
            buffer += delta.content
            # 在这里渲染到界面，不要解析

    parsed = json.loads(buffer)         # 只在流关闭之后解析
    for key in required:
        if key not in parsed:
            raise ValueError(f"missing field: {key}")
    return parsed
```

关键的是那个空行。界面渲染在循环里，解析和校验在循环外。在循环里调 `json.loads(buffer)` 的代码，除最后一块外每一块都会抛异常；哪个团队用增量 JSON 解析器去“修”这个问题，就离一次线上事故只差厂商的一个小怪癖。

**不流式，改成打字动画。** 发一个普通的阻塞调用。等完整响应。校验它。然后在客户端用打字机效果渲染，看起来跟流式一样。用户体感不变，你保住了校验保证。代价是对等待时间如实交代：白屏总时长现在是 TTFT 加上完整生成时间，不只是 TTFT。500 token 的回答，就是六秒转圈之后才出字。当输出要驱动下游动作、闪一下坏内容不可接受时用它，或者回答短到等待本来就不长时用它。

**用厂商的结构化输出加流式。** OpenAI 的 Structured Outputs（gpt-4o-2024-08-06 及之后）和 Anthropic 的对应功能（2025 年底加入）把 schema 直接烘进解码器。厂商在每一步生成时屏蔽非法 token，所以最终输出保证满足 schema。配合 `stream=True`，厂商 SDK 的辅助方法给出一个 `parsed` 对象，随着每个 JSON 值完成逐字段填进去。你既拿到流式体验，*又*拿到 schema 保证。OpenAI 称自家评测上 schema 合规率 100%，而旧的功能调用（function calling）方式在复杂 schema 上不到 40%。[^6]你拿不到的是部分校验：`parsed.address` 在整个字段流完之前是 `None`，流完才出现。

**管住输入，而不是管输出。** 开流之前先对用户提示词跑一个快的分类器。对抗性输入当场拒掉。这能以很低成本挡下多数越狱（jailbreak）尝试，但对良性输入上的幻觉毫无作用，所以它得跟另外三条中的一条搭配，不能替代它们。

## 流式拒答：2026 年的新坑

Claude 4 及之后的模型加了一个你必须显式处理的行为。安全分类器与生成并行跑。它若在流中途触发，生成停止，API 在一个 `message_delta` 事件里返回 `stop_reason: "refusal"`。[^4]不带任何解释文字。已经流出去的 token 早就到用户那儿了。

你的代码若没检查这个，用户看到的是：回答开始了，写了 50 个 token，然后……没了。停在半句。没有错误，没有提示。看起来像网络抽了一下，其实是模型在半路做的内容策略判断。

要做两件事。每一处 `message_delta` 都检查 `event.delta.stop_reason == "refusal"`（上面的循环就是这么写的）。重试之前清空对话上下文，因为 Anthropic 文档说得很明确：流式拒答之后沿用同一上下文，下一次回答也会被拒。[^4]还有一件收尾的事：那 50 个 token 是要花钱的。`message_delta` 事件的 `usage` 字段在拒答响应上也会报实际计费的 token 数。[^4]

> [!WARNING]
> **透传代理只要带 30 秒空闲超时，就会掐断推理模型的流。** 推理模型可能要 30 到 60 秒才吐出第一个 token，在负载均衡器看来这就是一条空闲连接。修法是客户端到厂商之间每一跳都调长读超时（推理档至少 120s），另外在思考阶段发 keep-alive 心跳，前提是你的厂商支持。

## 决策规则

在伸手写 `stream=True` 之前，按顺序走一遍：

1. **输出去哪儿？** 只进人的眼睛、不去别处，默认流式。要进代码、数据库、工具调用或另一个 LLM，那驱动动作的那部分别流：要么整个不流，要么只为展示而流，同时对缓存的那份做校验。
2. **审核必须在展示之前跑吗？** 医疗、金融、儿童安全、受监管内容：不流式。用打字动画那套。
3. **模型在推理模式吗？** 不要往界面上流。给加载态。第一个 token 要几十秒后才来，白屏 30 秒之后才开始的流式展示，比老老实实转圈更糟。
4. **既要流式又要 schema？** 用厂商的结构化输出配 `stream=True`。接受一点：字段级校验只在每个字段的 JSON 值完成之后发生，拿不到字符级的合法性。

聊天场景默认该流式。凡是字节要驱动行为的地方，默认都不该流式。这两类之间的那条线，就是本章大部分 bug 的来源。

## 参考文献

[^1]: Digital Applied Team, "AI Model Latency Benchmarks 2026: TTFT and TPS Data," Digital Applied, April 23, 2026. https://www.digitalapplied.com/blog/ai-model-latency-benchmarks-2026-ttft-throughput

[^2]: WHATWG, "HTML Living Standard, Section 9.2: Server-sent events," Living Standard (last updated June 1, 2026). https://html.spec.whatwg.org/multipage/server-sent-events.html

[^3]: OpenAI, "Streaming API responses," OpenAI Platform Docs, accessed June 2026. https://platform.openai.com/docs/guides/streaming-responses

[^4]: Anthropic, "Streaming refusals," Anthropic Developer Docs, accessed June 2026. https://docs.anthropic.com/en/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals

[^5]: Qwen3Guard team, "Qwen3Guard Technical Report," arXiv:2510.14276, October 2025. https://arxiv.org/html/2510.14276v1

[^6]: OpenAI, "Structured model outputs," OpenAI Platform Docs, accessed June 2026; "Introducing Structured Outputs in the API," OpenAI blog, August 6, 2024. https://platform.openai.com/docs/guides/structured-outputs
