---
title: "函数调用与工具调用"
description: "模型提出一次工具调用，你来执行，模型接着往下走：怎么定义工具 schema、怎么跑这个循环、怎么并行，以及怎么返回模型能据以行动的错误。"
slug: 03-function-and-tool-calling
part: 2
chapter: 3
difficulty: intermediate
prerequisites:
  - 00-calling-models
  - 02-structured-outputs
tags: [llm, http-api]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 函数调用与工具调用

模型从不自己执行任何东西。你把它接到一个 `get_weather` 函数上，它并不会去取天气；它返回一个 JSON 对象，意思是“请用 `location='Paris'` 调用 `get_weather`”，剩下的活由你的代码干。你运行函数、把结果递回去，模型就接着往下说。这一来一回就是全部机制，也是第 7 部分里每个 agent 脚下的地基。

第一天就让人困惑的地方在于：一个用户问题会触发*两次* API 调用。第一次问模型该做什么；第二次把结果交给模型，让它写回答。把这个循环装进脑子，本章剩下的都是细节。

```python
# 整体形状，伪代码。真正的代码在后面。
while True:
    response = model.call(messages, tools=tools)
    if not response.tool_calls:
        return response.text          # 模型说完了
    for call in response.tool_calls:
        result = run_locally(call.name, call.arguments)
        messages.append(result)       # 把结果喂回模型
```

## 模型读的是 schema

每家提供方都接受 JSON Schema（或它的子集）形式的工具定义，里面真正起作用的是三个字段：`name`、`description`，以及描述入参的 `parameters` 对象。description 决定模型会不会挑对工具——把它当成你代码库里最重要的文案。parameters 对象告诉模型哪些参数合法；在取值受限的字段上加一个 `enum`，能把胡编的取值砍掉一大截。

下面是同一个 `get_weather` 工具，分别写给 OpenAI 和 Anthropic，并排放着：

```python
# OpenAI Chat Completions：包在 {"type": "function", "function": {...}} 里
openai_tool = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Retrieves current weather for a given location.",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City and country, e.g. Paris, France"},
                "units":    {"type": ["string", "null"], "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location", "units"],
            "additionalProperties": False,
        },
        "strict": True,
    },
}

# Anthropic：扁平对象，用 input_schema 而不是 parameters
anthropic_tool = {
    "name": "get_weather",
    "description": "Retrieves current weather for a given location.",
    "input_schema": {
        "type": "object",
        "properties": {
            "location": {"type": "string", "description": "City and country"},
            "units":    {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["location", "units"],
    },
}
```

三处差异，足够让你踩坑：

- **外层包装。** OpenAI Chat Completions 把 schema 嵌在 `function` 下面；Anthropic 和 OpenAI Responses API 是平的。Gemini 把一组声明包在 `tools[].functionDeclarations[]` 里。
- **schema 的键名。** OpenAI 用 `parameters`；Anthropic 用 `input_schema`；Gemini 又回到 `parameters`。
- **严格模式。** OpenAI 的 `strict: true` 和 Anthropic 的严格工具用法都会启用受限解码（constrained decoding），保证参数一定符合 schema。代价是：每个属性都必须列进 `required`，`additionalProperties` 必须是 `false`。可选字段得写成可空联合类型（`["string", "null"]`）。[^1][^2]

工具定义在每次请求里都算输入 token。Anthropic 公布了按模型的额外开销：截至 2026 年 6 月，对 Claude Opus 4.8 来说，`tool_choice: auto` 时工具用法的系统提示词要 290 个 token，强制调用时要 410 个。[^2] OpenAI 不公布单个定义的开销数字，但同样把定义注入系统提示词。[^1] 两家都会缓存稳定的工具集，所以只要工具不随请求变，反复付的那部分成本基本被吸收掉了。单轮活跃工具最好少于 20 个；OpenAI 和 Google 都把这条标成软上限。[^1][^3]

两条建议，各家文档口径一致。第一，description 要写给看不到你源码的人：把 “fetches the user's order history given a user ID” 写成这样，比 “queries orders” 强得多。第二，总是一起跑的工具就合并成一个，模型已经知道的信息别让它再填一遍。[^1][^3]

## 执行循环

schema 是你传进去的东西，循环才是你真正要写的东西。每次迭代做五件事：把消息历史（含此前所有工具调用及其结果）发给模型，检查停止原因，派发模型请求的工具调用，把每个结果追加进历史，然后转下一轮。如果模型没要工具，你就退出并返回文本。

![一个五步循环图：模型提出工具调用，应用执行它，结果被追加进对话，循环继续直到模型返回最终答案](../../../ai-eng/part-2-working-with-model-apis/images/03-tool-calling-loop.png)
*循环的一轮。出口在第 2 步；其余每一步都回流进下一次请求。*

下面是能跑的代码，对着 OpenAI 的 Chat Completions API：

```python
import json

def execute_tool(name: str, args: dict) -> str:
    if name == "get_weather":
        return json.dumps({"temperature": 22, "unit": "celsius"})
    raise ValueError(f"Unknown tool: {name}")

def run_tool_loop(client, model, messages, tools, max_iterations=10):
    for _ in range(max_iterations):
        response = client.chat.completions.create(
            model=model, messages=messages, tools=tools,
        )
        msg = response.choices[0].message
        if not msg.tool_calls:
            return msg.content                          # 最终答案

        messages.append(msg)                            # 助手这一轮
        for tc in msg.tool_calls:
            try:
                args = json.loads(tc.function.arguments)
                result = execute_tool(tc.function.name, args)
            except Exception as exc:
                result = f"ERROR: {exc}"                # 见“把错误当数据”
            messages.append({
                "role": "tool", "tool_call_id": tc.id, "content": result,
            })
    raise RuntimeError("tool loop did not converge")
```

这段代码里有几个细节承担了全部工作。

OpenAI 的 `arguments` 字段到手时是 JSON 编码的字符串，不是 dict，用之前必须先 `json.loads`。Anthropic 的 `input` 和 Gemini 的 `args` 本身就是 dict；只有 OpenAI Chat Completions 发的是字符串形式。[^1][^2][^3] 忘了这一步，是工具调用代码里第二常见的 bug。

协议上的不变量很严：assistant 消息里的每个 `tool_call`，都必须在下次 API 调用之前配上恰好一个工具结果。如果你的派发循环抛了异常、漏掉一个结果，下一个请求会返回 400，报 “each tool_use must have a single result”。这就是为什么那段 `try/except` 无论如何都要追加*点东西*，哪怕执行已经失败。[^4]

还要注意 `max_iterations` 这道保险。一个永远报错的工具，配上一个永远想调它的模型，没有这道保险就会一直转下去。这是最低限度的安全网；真正的 agent 循环还会加 token 预算、墙钟超时和急停开关，[工作流与 agent](../../../ai-eng/part-7-agent-engineering/00-workflow-vs-agent.md) 那一章就建在这个骨架上。

这些循环放到规模化的 agent 系统里怎么组合——编排、扇出、状态持久化——[架构尺度的 AI 系统](../../hld/part-9-ai-ml-system-design/)给出了白板视角的那一层。

## 并行调用与打包规则

用户问一句“巴黎和东京的天气怎么样”，现代模型经常在一次响应里把*两个*工具调用都返回。你并发执行、收齐结果、再喂回去。做对了，多工具轮次的墙钟延迟直接减半；做错了，你会静默地教会模型以后干脆别用并行调用。

规则因提供方而异，而且两边搞反了都会把循环弄坏。在 Anthropic 的 API 上，规则精确得不能再精确，也是最难内化的一条：**同一批并行调用产生的所有 `tool_result` 块，必须放在一条 user 消息里回去。** 把它们拆成多条 user 消息，是 Anthropic 上生产环境工具调用代码最常见的 bug，他们的文档也明确点出来了。[^4] OpenAI 的 Chat Completions API 正好相反：每个结果都作为独立的 `role: "tool"` 消息回去，一个 `tool_call_id` 一条，就像本章前面那个派发循环干的。把 Anthropic 的打包规则用到 OpenAI，或者把 OpenAI 一条结果一条消息的习惯用到 Anthropic，两种都是 400。

```python
import asyncio, json

async def run_one(tool_id: str, name: str, args: dict) -> dict:
    try:
        result = await execute_tool_async(name, args)
        return {"type": "tool_result", "tool_use_id": tool_id, "content": result}
    except Exception as exc:
        return {
            "type": "tool_result", "tool_use_id": tool_id,
            "is_error": True, "content": str(exc),
        }

async def handle_parallel_batch(tool_calls):
    results = await asyncio.gather(*[
        run_one(tc["id"], tc["name"], tc["input"]) for tc in tool_calls
    ])
    # Anthropic：所有结果放进一条 user 消息。不是三条。
    return {"role": "user", "content": results}
```

为什么拆分这么要紧？消息历史就是模型学习会话模式的素材。如果它看到 `assistant(2 tool_uses) -> user(result_1) -> user(result_2)`，两件事会同时发生：API 直接拒掉这个结构错误，而且后续轮次里模型会越来越偏向单次调用，因为历史就长这样。把结果打包回去，并行调用才能保持顺畅。[^4]

需要的时候也可以关掉并行调用。OpenAI 在请求里收 `parallel_tool_calls: false`；Anthropic 在 `tool_choice` 对象里收 `disable_parallel_tool_use: true`。[^1][^2] 什么时候该拉这个开关：工具有副作用、必须以确定顺序执行（同一资源上先创建再更新），或者下游基础设施扛不住并发压力。只读的取数和彼此独立的查询，让它开着就行。

> [!WARNING]
> **把并行结果拆成多条 user 消息，出事之前一直很安静。** 第一个症状是模型“不再用并行调用”，几周之后才是某个轮次里结构终于崩掉、返回 400。想早点发现，就统计每条含工具调用的 assistant 消息里 `tool_use` 块的平均个数；任何真在做扇出的 agent，这个数都该高于 1.0。[^4]

## 把错误变成模型能接着干活的数据

工具调用失败不是你抛出的异常，而是你返回的一份数据。模型把错误字符串当成对话的一部分读，写得好的错误能让它重试、换工具，或者把问题摊给用户——这几件事，你的派发循环一崩就全做不到了。

两种格式：

- **Anthropic** 在 `tool_result` 块上有一等公民的 `is_error: true` 标志。模型认得这个标志，会显式地对失败做推理。[^2][^4]
- **OpenAI 和 Gemini** 没有错误标志；你把错误当成结果字符串返回，模型从上下文里自己判断。[^1][^3]

错误的文字比标志更重要。“404 Not Found” 这种通用码让模型无事可做；具体、指向动作的消息才能让它恢复。这套设计纪律再往深走——命名约定、错误分类、注册表模式——在[工具设计](../../../ai-eng/part-7-agent-engineering/03-tool-design.md)里；现在先比较这两条：

```python
# 没用
return "ERROR: 404"

# 模型能照着这个干活
return "ERROR: user 'alice' not found. Valid IDs are integers in 1000-9999. Try get_user(id=1234)."
```

这样打开的自恢复能力相当强。Anthropic 文档里就记了并行调用带隐藏依赖时的用法：`read_file("report.md")` 和 `summarize(file=...)` 一起派发出去，读文件失败了，就带着 `is_error: true` 返回那个自然的错误（`"cat: report.md: No such file or directory"`）。模型看出依赖关系，重排调用顺序，下一轮先重发 `read_file`。[^4] 你不需要预防性地改成串行派发；让循环从一条有用的错误信息里自我纠正就好。

现实里也有记录。`claude-code` 仓库在 2025 年把 “parallel tool calls cascade-fail when one fails” 记成 issue #22264，修法正是上面这个 `is_error` 恢复模式。[^5] 如果你的代码只会抛异常、从不返回结果，等于绕开了整个恢复机制，API 只会把下一个请求拒掉。

顺带提一个 SDK 的坑：`content` 期望的是字符串（图片和文件结果则是结构化的内容数组）。直接返回一个裸 Python dict，在不同 SDK 版本里被强转的结果并不一致。结构化结果离开派发器之前，一律先 `json.dumps`。

## 参考文献

[^1]: OpenAI, "Function calling," OpenAI API Docs, https://platform.openai.com/docs/guides/function-calling (accessed June 2026).
[^2]: Anthropic, "Tool use with Claude, overview," Anthropic Docs, https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview (accessed June 2026).
[^3]: Google, "Function calling with the Gemini API," Google AI for Developers, https://ai.google.dev/gemini-api/docs/function-calling (last updated 2026-06-10).
[^4]: Anthropic, "Parallel tool use," Anthropic Docs, https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use (accessed June 2026).
[^5]: GitHub, "parallel tool calls cascade-fail when one fails," anthropics/claude-code issue #22264, https://github.com/anthropics/claude-code/issues/22264 (opened 2025).
