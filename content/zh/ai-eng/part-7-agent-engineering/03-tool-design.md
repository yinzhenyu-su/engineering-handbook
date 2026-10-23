---
title: "工具设计"
description: "智能体工程的基本功：让非法调用无法发生的 schema、模型读不歪的描述、模型能据以恢复的错误，以及恰当的工具数量。"
slug: 03-tool-design
part: 7
chapter: 3
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/03-function-and-tool-calling
  - 02-the-agent-loop
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 工具设计

Anthropic 上线网页搜索工具后，上线后的评测暴露出一个奇怪的现象：Claude 在悄悄给每个查询加上 `2025`。搜索结果偏向近期内容，检索质量下降，而没有人要求模型这么做。修复方式只是给工具描述改了一段话。没有动任何代码。[^1]

这就是本章要讲的全部内容。工具的 schema 不是给编译器看的类型签名，而是给智能体看的提示词。模型通过三个界面认识你的工具，顺序是：决定调用之前读到的描述、约束它能输出什么的 schema、调用失败后返回给它的错误字符串。这三个界面都承重。搞错了，错误就会在每一轮、每个用户身上复利，就像那个 `2025` bug。

[函数与工具调用](../part-2-working-with-model-apis/03-function-and-tool-calling.md) 一章讲了线格式和循环本身。本章讲叠加在其上的设计纪律。有四件事要做对，按投入产出比从高到低排：schema、描述、错误信息、粒度。

## 让非法调用无法发生

schema 是你唯一能拿到的机械保证。其余都只是模型可以无视的散文。用上这个保证。

Anthropic 和 OpenAI 都提供 strict 模式，把你的 JSON Schema 编译成采样语法（sampling grammar），模型就无法输出那些构成非法调用的 token。设上 `strict: true` 之后，`passengers: int` 永远不会以 `"two"` 或 `"2"` 的形式到达；API 在采样时就拒掉这些 token，而不是等到校验阶段。[^2][^3] 代价是一次性的：schema 在首次使用时编译，然后缓存 24 小时。[^2] 收益是整类“解析失败再重试”的故障从你的 trace 里消失了。

strict 模式带两条要求，每个迁移过来的团队都会踩。每个 object 都必须声明 `additionalProperties: false`，每个属性都必须出现在 `required` 里。可选字段要保持可选，做法是把它写成可空联合类型（`["string", "null"]`），但仍然列进 `required`。[^2][^3] 两条里漏一条，API 会在第一次调用时以 400 拒掉这个 schema。

除了 strict 模式，schema 还是你编码约束的地方——那些否则只能写在描述散文里的约束：

- **参数名要无歧义（unambiguous）。** `user_id` 胜过 `user`。`max_results` 胜过 `n`。Anthropic 的文档专门点名过这件事：歧义参数会造成静默误用。[^4]
- **封闭集合用 enum。** `section` 字段写成 `enum: ["policies", "products", "procedures"]`，"marketing" 就变得无法表示。语法不会采样的值，模型也编不出来。
- **给数值定界。** 在 `max_results` 上加 `minimum: 1, maximum: 10`，省掉一次模型本来还得去读的可恢复错误。
- **避免自相矛盾的形状。** `toggle_light(on: bool, off: bool)` 是 OpenAI 举的经典反例：这种 schema 允许模型输出 `{on: true, off: true}`。一个 enum 字段永远胜过两个布尔值。[^3]

下面这个定义把四条都做到了：

```python
tool_definition = {
    "name": "search_documents",
    "description": (
        "Full-text search across the internal knowledge base. "
        "Use this when the user asks about company policies, procedures, "
        "or product documentation. Do NOT use for real-time data or "
        "live prices. Returns up to 10 results ranked by relevance."
    ),
    "strict": True,
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Natural language question, e.g. 'What is the refund policy?'",
            },
            "max_results": {
                "type": "integer",
                "minimum": 1,
                "maximum": 10,
                "description": "Number of results to return. Default 5, max 10.",
            },
            "section": {
                "type": ["string", "null"],
                "enum": ["policies", "products", "procedures", None],
                "description": "Optional section filter. Pass null to search all sections.",
            },
        },
        "required": ["query", "max_results", "section"],
        "additionalProperties": False,
    },
}
```

这是及格线。所有生产工具默认开 strict 模式；只有当你确实需要无法预先规定的动态 schema 时才例外。在我们两年的智能体 trace 里，还没见过这种例外划算过。

## 描述才是主力

strict 模式告诉模型它能输出什么。描述告诉它到底要不要调这个工具，而这个决定比参数更重要。Anthropic 的工程团队在用自己的评测套件重建 Slack 和 Asana 的 MCP 工具之后，把描述质量称为“工具表现里最最重要的因素”，排在 strict 和合并之上。[^1]

一份好的描述按这个顺序覆盖五件事：

- **它做什么**，一句话平实说清。
- **什么时候用它**，包括模型可能漏掉的场景。
- **什么时候不要用它**，点名负责相邻工作的兄弟工具。
- **每个参数是什么意思**，带格式提示，格式不明显的地方给一个示例值。
- **它返回什么**，以及它明确不返回什么。

每个工具至少写三到四句；触发条件不明显或已知有误用模式的，写八到十二句。[^4] 是的，这要花上下文 token。工具定义在每次请求上都要计费：光是工具调用那套脚手架就大约 290 到 500 个 system prompt token，再加上每份描述的正文。[^5] 这些 token 你付定了，不管它们有没有教会模型什么，所以就让它们教。

![并排的两份工具描述：左边是一句话描述配一个困惑的模型，右边是结构化的五段式描述配一个笃定的模型](../../../ai-eng/part-7-agent-engineering/images/03-description-quality.png)
*同一个工具，两份描述。左边的模型只能猜，右边的不用猜。*

SWE-bench 的结果把“把描述写好”的回报讲得很具体。Anthropic 在 2025 年 1 月公布 49% 的 SWE-bench Verified 成绩时（发表时为最好水平，此前是 45%），智能体只跑了两个工具：一个 Bash 工具，一个 `str_replace_editor` 工具。schema 极简，描述不简。Bash 工具的描述用平白英文把不明显的约束写死：“the contents of the `command` parameter does NOT need to be XML-escaped. You don't have access to the internet via this tool. State is persistent across command calls. Please avoid commands that may produce a very large amount of output.” 团队写道，他们在这两个工具的描述和规范上花了很大力气，还专门测试过模型可能误解规范的每一种方式。[^6]

最后那句就是纪律本身。工具描述值得你像对待 system prompt 那样迭代：写出来，跑评测，读失败案例，留意模型做了描述没有禁止的事，然后收紧措辞。网页搜索的 `2025` bug 就是这么抓到的。SWE-bench 的 Edit 工具上 `path` 参数的小修也是：它最初接受相对路径，团队发现智能体在 `cd` 之后会犯迷糊，于是改写成 `"Absolute path to file or directory, e.g. /repo/file.py"`。[^6]

判断一份描述是否到位，标准很简单：你能不能通过大声念一遍描述、指出那个缺口，来回答“模型为什么没调这个工具”或者“它为什么用错方式调了”。

## 模型能据以行动的错误

失败的调用不是你要抛出的异常，而是一段模型马上要读、要拿来推理的字符串。为这个读者写。

工具分发代码的默认失败方式是把原始异常直接透出去：`TypeError: argument 'max_results' must be int, not str`。模型拿这个做不了什么。它常常用同样的输入重试，或者用语法不同但语义相同的输入重试，或者干脆放弃工具、编造一个结果。我们见过智能体卡在单个坏参数上五轮重试，烧 token、打穿预算，而一条写得好的错误信息本可以在第二轮就把它解开。

可恢复错误（recoverable error）的配方有三部分：点名参数、给出合法范围或格式、给一个改对的调用示例。

```python
# Useless
return "ERROR: invalid argument"

# Useless and worse, because the model can't tell what to fix
return "TypeError: int() argument must be a string..."

# The model can act on this
return (
    "Error: max_results=15 exceeds limit of 10. "
    "Retry with max_results=10 or lower. "
    "Example: search_documents(query='refund policy', max_results=10, section=null)"
)
```

产出这类错误的分发写法是这样：

```python
import json

def execute_tool(name: str, args: dict) -> dict:
    if name == "search_documents":
        query = args.get("query", "")
        max_results = args.get("max_results", 5)

        if not query.strip():
            return {
                "is_error": True,
                "content": (
                    "Error: 'query' is empty. Provide a non-empty natural "
                    "language question. Example: search_documents("
                    "query='What is the return policy?', max_results=5, section=null)"
                ),
            }
        if max_results > 10:
            return {
                "is_error": True,
                "content": (
                    f"Error: max_results={max_results} exceeds limit of 10. "
                    "Retry with max_results=10 or lower."
                ),
            }
        return {"is_error": False, "content": json.dumps(do_search(query, max_results))}

    return {"is_error": True, "content": f"Unknown tool: {name}."}
```

这个结构对应 Anthropic 和 MCP 各自定义的协议：错误装在工具结果里传回，而不是作为抛出的异常，并带一个标志（Anthropic 是 `is_error: true`，MCP 是 `isError: true`），告诉模型这个结果是待恢复的失败，而不是待消费的数据。[^7][^8] OpenAI 的协议没有这个标志，但同样的模式照样适用：模型从上下文里读错误字符串。

两个改法，在长时间运行的智能体里能自己把成本赚回来：

- **截断要带指令，不要硬切。** 工具返回的内容超过 token 预算时，截断并在末尾追加 `"Truncated after N tokens. Use offset and limit parameters to retrieve subsequent pages."` 这把一堵墙变成一架梯子。[^1]
- **有副作用的错误要区别对待。** 对幂等读操作，“用修正后的输入重试”是个好默认值。对发邮件或刷卡的这类有副作用的工具，错误信息应该要求模型先跟用户确认再重试，而不是直接重试。错误字符串就是你写这条策略的地方。

可恢复错误确实要多花几轮，也就要多花 token。对便宜的工具，这笔交易压倒性地划算。对昂贵的工具，错误信息本身既是控制面，也是调试日志。

## 少数信息量大的工具胜过二十个单薄的

最后一个设计维度是粒度（granularity）：模型每轮从多少个工具里挑，以及每个工具干多少事。往单薄工具那边滑是重力的方向。每个后端接口都变成一个工具，每个微服务配一个包装，六个月后你的智能体每轮要面对三十二个函数定义。可见工具超过二十个左右，工具选择的准确率就会明显下降，OpenAI 的文档也正是因此把这个数说成软上限。[^3] Anthropic 没有公布硬上限，但建议每个工作流用“少数几个想清楚了的工具”。[^1]

单薄工具失败的原因，不是模型读不了三十二份描述，而是单薄工具逼模型去干本该由应用干的活。像 `get_customer_by_id` → `list_transactions` → `list_notes` 这样的三段调用，把模型放在三次往返的中间，为所有中间上下文付费。每次调用都带自己的头部噪声；每个结果都要被总结进下一次调用的参数里；每一轮都有模型跑偏的风险。

把这三个换成一个 `get_customer_context(customer_id, response_format)`，智能体就只做一次决定，而不是三次。Anthropic 直接量过返回结构这一块：他们内部的 Slack 工具加上 `response_format: "concise"` 模式后，从每次调用 206 个 token（完整 ID、全部元数据）降到 72 个 token（语义化名字、去掉 ID）。同样的信息，少了三分之二 token。[^1]

合并规则，具体说：

- **同一工作流里总是被顺序调用的工具，合并。** `get_customer + list_transactions + list_notes` 变成 `get_customer_context`。`list_users + list_events + create_event` 变成 `schedule_event`。[^1]
- **概念上不同的操作不要合并。** `read` 工具和 `write` 工具分开；搜索工具和变更工具分开。分界线在于读写有不同的故障模式、不同的恢复语义、不同的审计要求。
- **跨服务时用命名空间。** 前缀式命名（`crm_search_contacts`、`support_search_contacts`）在区分重叠职责上，比任何写得一模一样的描述都强。[^1][^3]
- **可见工具超过二十个，就延迟加载。** OpenAI 的 `tool_search` 和等价的条件加载模式，让模型只在需要时拉进一块工具面，而不是每一轮为全部工具付费。[^3]

反对意见是真实存在的，值得写出来。合并过头的工具会长出臃肿的参数面（`schedule_event` 悄悄处理时区解析、冲突检测、会议链接创建），要把它描述得无歧义，比三个小工具更难。解法不是固定数量，而是：合并应当跟随人类真实的工作流，而不是把不相关的动作人为凑成一组。如果你给不出合并后工具的三句话干净描述，那就是合并过头了。

这也是工具设计这条纪律在上下文工程上闭环的地方。每个工具定义都是每次请求的固定税。二十个描述冗长的工具，在用户还没敲一个字之前就能烧掉 2,000 到 5,000 个输入 token。[^5] 描述上的精简、工具面上的合并、返回上的 `response_format` 模式，是同一根杠杆的三个方向：让模型的工作记忆装满用户的问题，而不是你的服务目录。[上下文工程](../part-5-context-engineering/04-tool-context-and-metadata.md) 讲的是预算那一侧；工具设计是你把预算花好的地方。

这些工具面如何组合成带共享注册表、沙箱和审计的智能体平台，架构尺度上的看法见 [架构尺度上的 AI 系统](../../hld/part-9-ai-ml-system-design/)。

## 参考文献

[^1]: Ken Aizawa et al., "Writing effective tools for agents, with agents," Anthropic Engineering, Sep 11, 2025. https://www.anthropic.com/engineering/writing-tools-for-agents
[^2]: Anthropic, "Strict tool use," Anthropic Docs, accessed June 2026. https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/strict-tool-use
[^3]: OpenAI, "Function calling," OpenAI Platform Docs, accessed June 2026. https://platform.openai.com/docs/guides/function-calling
[^4]: Anthropic, "Define tools," Anthropic Docs, accessed June 2026. https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/define-tools
[^5]: Anthropic, "Tool use with Claude, Overview (Pricing)," Anthropic Docs, accessed June 2026. https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
[^6]: Erik Schluntz, "Raising the bar on SWE-bench Verified with Claude 3.5 Sonnet," Anthropic Engineering, Jan 6, 2025. https://www.anthropic.com/engineering/swe-bench-sonnet
[^7]: Anthropic, "Tool use with Claude, overview," Anthropic Docs, accessed June 2026. https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview
[^8]: Model Context Protocol, "Tools," MCP Specification 2025-06-18, accessed June 2026. https://modelcontextprotocol.io/specification/2025-06-18/server/tools
