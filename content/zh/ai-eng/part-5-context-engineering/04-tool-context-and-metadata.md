---
title: "工具上下文与元数据"
description: "模型没有时钟，没有角色意识，也没有区域设置的概念。你注入的元数据、以及你描述工具的方式，能挡住好几类自信地答错的回答。"
slug: 04-tool-context-and-metadata
part: 5
chapter: 4
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/03-function-and-tool-calling
  - 00-context-is-more-than-the-prompt
  - 02-compression-and-context-budgets
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 工具上下文与元数据

用户对你的助手说：“把上周的收据调出来。”模型返回的是 2023 年 10 月的收据。你的代码没问题，检索没问题。模型只是不知道现在是哪一周。

一个 bug 就说清了本章。模型没有时钟，不知道提问的人是谁，没有区域设置的概念，对每个工具到底做什么也只有模糊印象。凡是你没摆在它面前的东西，它只能靠训练数据的先验去猜，而那些先验可能已经过期二十九个月。[^1] 生产环境里绝大多数“答得自信却是错的”，都能追到这类缺口上，而不是模型推理能力差。

[函数调用与工具调用](../part-2-working-with-model-apis/03-function-and-tool-calling.md)讲的是这个循环的机制：schema 进去，JSON 参数出来，结果回传。本章讲另一半：模型需要知道关于自己的工具、用户和环境的哪些信息，才能选对工具、调用正确，并给出跟现实对得上的答案？

## 决定路由的是描述，不是 schema

多数工程师看待工具定义，跟他们看待一个 TypeScript 类型一样：JSON Schema 才是真契约，description 只是文档。模型的理解正好相反。schema 约束的是模型*决定调用之后*的参数；要不要调这个决定，几乎完全由自然语言写的 description 决定。

Anthropic 自己的说法很直白：description 是“工具表现中远居首位的影响因素”。[^2] OpenAI 用另一种说法讲了同一件事。[^3] 2025 年 1 月 Anthropic 公布 SWE-bench Verified 成绩背后的工具设计时，最抢眼的数字是 Claude 3.5 Sonnet 只靠两个工具拿到 49%，超过了此前用更庞大自定义脚手架的 45%。[^4] 起作用的是什么？是 `bash` 上那份长而带主见的 description，以及一个字符串替换编辑器——它把团队在评测轨迹里见过的每一种误用都提前堵住了。schema 设计是次要的。

同一批工作里还有个真实例子：Claude 的网页搜索工具刚上线时，评测记录显示它总往用户的查询后面追加 `2025`，把每条结果都带偏了。修法是在 description 里加一段话，告诉模型别这么干。[^5] 没改代码，没重新训练。一段话。

所以：写 description 要像给一个看不到你代码的新同事做交接。讲清工具做什么、什么时候该用、什么时候*不*该用，以及每个参数（包括任何冷门术语）的格式。OpenAI 文档把这条标准说得很具体：“只给实习生你给模型的那点信息，他能正确使用这个函数吗？”[^3] 如果他会追问一个问题，那个问题的答案就该写进 description。

由此掉出三条规矩：

- **参数名要能自证含义。** `start_date_iso8601` 胜过 `start_date`，`customer_id` 胜过 `customer`。description 没交代的地方，模型就靠名字猜。
- **别让模型填你已经知道的参数。** 应用从会话里就有 `tenant_id`，那就在代码里传。模型不得不自己编的每个参数，都是它能幻觉出来的参数。
- **把相关操作收进一个 `action` 枚举。** `create_pr`、`review_pr`、`merge_pr` 三个工具在代码里很整齐，却逼着模型去分辨三份几乎一样的 description。一个 `pr_action` 工具，`action: "create" | "review" | "merge"`，选起来更可靠。[^5]

## 模型对世界一无所知的部分

开头那个收据 bug，属于一整类问题里最容易的那种。模型没有内部时钟。它推理“今天”时，用的是训练末期占主导的那些日期形状的 token——对 GPT-4o 来说就是 2023 年 10 月。[^1] 到 2026 年初，这是二十九个月的差距。ChatGPT 和 Claude.ai 靠每次对话都往系统提示里注入当前日期，把这个问题糊过去了；API 不会。你自己不注入，就等于把这段差距发布出去了。

修法只有一行，但写哪一行很关键：

```python
import datetime

# 对：只给日期精度，ISO 8601，UTC。24 小时内保持稳定。
today = datetime.datetime.now(datetime.UTC).date().isoformat()
system = f"Today's date is {today} UTC.\n\n" + base_instructions

# 错：完整 datetime。每个请求都不同。把 prompt 缓存打穿。
# now = datetime.datetime.now(datetime.UTC).isoformat()
```

那条注释不是摆设。有个工程团队发现 LLM 延迟从几秒跳到五十多秒，追下去发现是 `Current date and time: ${dateTime}` 这样一行按秒精度格式化的代码。[^1] 每个请求都产生独一无二的系统提示前缀，供应商的 prompt 缓存命中率直接归零，于是每次调用都按全价输入 token 计费，还要付完整的 prefill 延迟。日期精度让前缀在二十四小时内保持不变，缓存一直是热的，账单也正常。[prompt 缓存](../part-2-working-with-model-apis/07-prompt-caching.md)那一章讲了稳定前缀为什么重要；这里是最值得保持稳定的地方之一。

还有两点是关于日期*放在哪*的。放在系统提示的最前面、指令之前，让模型在读到任何可能跟日期有关的内容之前，就先锚定时间推理。[^1] 而且要放的是*日期*；如果你确实需要一天中的时刻或用户的本地时区，把这些放进第一条用户消息，那里每轮都变，也污染不到被缓存的前缀。

长时运行的 agent 需要第三种方案。一个会话在晚上 11:45 打开、缓存了 `Today is April 19, 2026`，到 12:15 再服务请求时，算“明天”就会算错。这个 bug 只在午夜前后两小时的窗口里发作，落在缓存 TTL 跨过的那条时区边界上，而且你在上班时间测试根本复现不出来。[^1] 对可能跑几个小时的 agent，给模型暴露一个小工具，它需要时自己调：

```python
def get_current_time(timezone: str) -> str:
    """Return the current time in the requested IANA timezone, e.g. 'America/New_York'.
    Use this whenever the user asks about today, tomorrow, or anything time-relative,
    if the session has been running for more than an hour."""
    import datetime
    from zoneinfo import ZoneInfo
    return datetime.datetime.now(ZoneInfo(timezone)).isoformat()
```

模型会读这段描述，在需要新鲜时间时调用它。系统提示里注入的日期覆盖常见情况，这个工具覆盖长尾。

## 提问的人是谁，身在哪里

同样的逻辑适用于用户。模型分不清对面是管理员还是只读用户、是医生还是小孩、是期待 `19.04.2026` 的德国用户还是期待 `04/19/2026` 的美国用户。你没告诉它的，它就按训练里占多数的模式挑一个。修法是在系统提示顶部放一小段结构化的元数据：

```python
from dataclasses import dataclass

@dataclass
class RequestContext:
    user_role: str    # "admin" | "editor" | "viewer"
    locale: str       # BCP-47, e.g. "en-US", "de-DE"
    timezone: str     # IANA, e.g. "America/New_York"

def build_system_prompt(base: str, ctx: RequestContext) -> str:
    today = datetime.datetime.now(datetime.UTC).date().isoformat()
    lines = [
        f"Today's date is {today} UTC.",
        f"User locale: {ctx.locale}. Timezone: {ctx.timezone}.",
        f"User role: {ctx.user_role}.",
    ]
    if ctx.user_role == "viewer":
        lines.append("You have read-only access. Do not suggest write, edit, or delete actions.")
    elif ctx.user_role == "admin":
        lines.append("You have full administrative access including deletion and config changes.")
    lines.append("")
    lines.append(base)
    return "\n".join(lines)
```

原则是只注入那组“缺了就会改变模型答案”的事实，而且是最小集合。日期永远算一个。任何有权限概念的系统里，角色都算一个。只要模型要格式化数字、货币或日期，或者要作跟时间有关的陈述，区域设置和时区就算。剩下的都是填充物，而填充物就是你在每个请求上白付的 token。

![一条横向的 LLM 请求时间线，三支彩色箭头指向不同的注入点：日期和角色进入系统提示，一天中的时刻进入第一条用户消息](../../../ai-eng/part-5-context-engineering/images/04-metadata-injection-timeline.png)
*稳定的元数据放系统提示的前缀里，每轮都变的元数据放用户消息里。搞错了，缓存命中率就崩。*

有一点容易把人坑住。注入 `user_role: viewer` 只是引导模型，不强制执行任何东西。一个被告知对面是只读用户的模型，只要 `delete_order` 还在它的工具数组里，再来一段足够有说服力的提示注入，它照样会调。把 LLM 当成授权层，是那种最后会写进事故复盘的错误。[^6]

修法是纵深防御：注入角色来引导模型，*同时*在发请求前过滤工具数组，让只读用户压根看不到破坏性工具：

```python
ROLE_ALLOWED_TOOLS = {
    "viewer":  {"read_orders"},
    "editor":  {"read_orders", "cancel_order"},
    "admin":   {"read_orders", "cancel_order", "issue_refund"},
}

def filter_tools_by_role(tools, role):
    allowed = ROLE_ALLOWED_TOOLS.get(role, set())
    return [t for t in tools if t["function"]["name"] in allowed]
```

元数据注入引导推理，工具过滤在结构上落实约束。真正的授权仍然在你的 API 和数据库里，但这两层配合起来，能在角色混淆 bug 到达模型之前就干掉最难堪的那一类。

## 工具数量的税

你注册的每份工具定义都住在每个请求的系统提示里，不管模型用不用。一个单参数的简单工具大约烧 96 到 150 个 token；一个 28 参数的复杂工具要吃掉约 1,633 个。[^7] 接上官方 GitHub MCP 服务器，它自带 93 个工具，用户还没敲一个字，你已经花掉 55,000 个 token，约占 200K 上下文窗口的 21%。[^8][^9] 接十个 MCP 服务器、每个十五个工具，就是大约 75,000 个 token 的固定税。[^10]

这个成本真实，但还兜得住。准确率上的代价更糟。截至 2026 年的独立基准显示：工具在 50 个左右时，前沿模型仍有 84% 到 95% 的选对率；到 200 个，区间碎成 41% 到 83%，因模型而异；到 740 个，准确率塌到 0 到 20%。[^11] 工具更多不等于能力更强。过了某个点，更多工具只是更多混乱。

OpenAI 给的软上限是“一轮开始时可用工具少于 20 个”。[^3] 这个锚点是对的。越过去之后，有四种可以叠加的做法：

- **按角色或任务过滤工具**，如上所述。不同用户确实需要不同工具时，这是最干净的收益。
- **工具搜索。** 只暴露一个 `tool_search` 工具，模型可以调它按需发现并加载 schema。Spring AI 的实现把工具 token 省下 34% 到 64%，在 OpenAI、Anthropic、Gemini 上都成立。[^12] 你为加载一个工具多付一轮延迟，换来其他每一轮都免掉这份税。
- **对工具块做 prompt 缓存。** 工具定义跨请求稳定，正是 prompt 缓存为之而生的那种又长又重复的前缀。缓存写入比基础输入价格贵约 25%，第二次命中就回本。[^13] 只是别在工具块上面放每请求都变的时间戳，否则这套全白搭。
- **上下文编辑。** 长 agent 运行中，模型用完的 `tool_result` 块就从历史里丢掉。这跟[压缩与上下文预算](02-compression-and-context-budgets.md)里讲的是同一个思路，只是专门作用在工具输出上。

开局建议：初始集合压在 20 个以内，第一天就把工具块的 prompt 缓存打开，真的需要更多时再上工具搜索。

## MCP 注解能做什么，不能做什么

如果你的工具来自 [Model Context Protocol](../../../ai-eng/part-7-agent-engineering/04-mcp-model-context-protocol.md)，工具对象上还会看到第五个字段：`annotations`。从 2025-03-26 的规范修订起，一个 MCP 工具可以声明四个行为提示：[^14]

```python
# 摘自 MCP 规范。默认值是故意取悲观的。
annotations = {
    "readOnlyHint":    False,  # 默认：假定工具会修改状态
    "destructiveHint": True,   # 默认：假定这次修改是破坏性的
    "idempotentHint":  False,  # 默认：假定重试不安全
    "openWorldHint":   True,   # 默认：假定工具会跟外部系统打交道
}
```

客户端可以用它们来驱动交互。任何非 `readOnly` 的工具，调用前弹个确认框。对幂等的工具问一次然后记住。当会话凑齐了[致命三要素](../../../ai-eng/part-9-security-and-governance/01-agent-security-and-the-lethal-trifecta.md)的三条腿——访问私有数据、摄取不可信内容、对外通信——就把带 `openWorldHint` 的工具标出来。[^15]

MCP 规范明确写了：注解是提示，不是契约。一个不可信的服务器可以声称 `readOnlyHint: true`，照样删你的文件。[^14] 采用情况也印证了这一点：截至 2026 年 3 月，只有约 17% 的 GitHub MCP 用户开了只读模式。[^14] 注解用来驱动客户端的确认流程，别拿来当安全边界。真正的隔离在传输层认证、沙箱和出口管控里，那是[Agent 安全与致命三要素](../../../ai-eng/part-9-security-and-governance/01-agent-security-and-the-lethal-trifecta.md)的地盘。

## 工具返回的内容也是上下文

工具调用的输出会直接进入模型的工作记忆。一份臃肿、塞满 UUID 的响应，在它之后的每一轮都在浪费 token，还给了模型更大的幻觉面积。Anthropic 对内部 Slack 和 Asana MCP 服务器的评测发现：把不透明的内部 ID 解析成人类可读的名字，在检索任务上可测量地降低了幻觉率；[^5] 而一个让模型在 `concise` 和 `detailed` 之间选择的 `response_format` 枚举，把一条 Slack 线程响应从 206 个 token 压到 72 个，约为原来的 35%，且在留出测试集上没有准确率损失。[^5]

工具输出有三条规矩：

- **返回模型推理需要的东西，而不是数据库碰巧存了什么。** 名字胜过 UUID，状态枚举胜过裸整数码。MIME 类型和内部引用都剥掉。
- **长响应要分页，给个合理的默认值。** Claude Code 把工具输出上限压在 25,000 个 token。[^5] 截断时要在响应里说明，并告诉模型怎么取更多：“showing 50 of 200 results; pass `page=2` for the next batch.”
- **错误信息要能指导行动。**“ERROR: 404”等于什么都没告诉模型。“ERROR: user 'alice' not found. Valid IDs are integers between 1000 and 9999. Try `get_user(id=1234)`.”能让模型拿一个真实的参数重试，或者升级给用户。错误分类的更多细节在[工具设计](../../../ai-eng/part-7-agent-engineering/03-tool-design.md)那一章。

这三条规矩说到底都是本章已经出现过四回的那个思路，只是换了块表面。模型只知道摆在它面前的东西。你没放在那儿的，它就去猜，而且猜得十足自信。工具描述、环境元数据、工具清单本身、工具返回的响应，是同一份上下文的四个通道。每个通道挡住的是不同的一类“自信地答错”。

## 参考文献

[^1]: Tian Pan, "Temporal Context Injection: Making LLMs Actually Know What Day It Is," TianPan.co, April 20 2026, https://tianpan.co/blog/2026-04-20-temporal-context-injection-llm
[^2]: Anthropic, "Define tools," Anthropic API Docs, https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/define-tools
[^3]: OpenAI, "Function calling," OpenAI API Docs, https://platform.openai.com/docs/guides/function-calling
[^4]: Erik Schluntz, "Raising the bar on SWE-bench Verified with Claude 3.5 Sonnet," Anthropic Engineering Blog, January 6 2025, https://www.anthropic.com/engineering/swe-bench-sonnet
[^5]: Ken Aizawa et al., "Writing effective tools for agents, with agents," Anthropic Engineering Blog, September 11 2025, https://www.anthropic.com/engineering/writing-tools-for-agents
[^6]: Oracle Developers Blog, "How to Enforce Role-Based Data Access in AI Applications with Oracle Deep Data Security," 2026, https://blogs.oracle.com/developers/how-to-enforce-role-based-data-access-in-ai-applications-with-oracle-deep-data-security
[^7]: Tianpan.co, "Why Your Agent Breaks at 30 Tools," April 2026, https://tianpan.co/blog/2026-04-13-tool-explosion-problem-agent-tool-selection-at-scale
[^8]: Piotr Hajdas, "MCP Tools Token Allocation Report" (GitHub gist), August 2025, https://gist.github.com/ghuntley/1fe34212c515a59ea2a9963b5f2386bf
[^9]: Simon Willison, "too many model context protocol servers and LLM allocations on the dance floor," August 22 2025, https://simonwillison.net/2025/Aug/22/too-many-mcps/
[^10]: getunblocked.com, "A Measured Guide to Context-Window Bloat," 2025-2026, https://getunblocked.com/blog/mcp-tool-overload/
[^11]: Tianpan.co, "Your Tool Catalog Is a Power Law and You're Optimizing the Long Tail," April 2026, https://tianpan.co/blog/2026-04-27-tool-catalog-power-law-hot-cold-partition
[^12]: Mario Tzolov, "Achieving 34-64% Token Savings with Spring AI's Dynamic Tool Discovery," Spring Blog, December 11 2025, https://spring.io/blog/2025/12/11/spring-ai-tool-search-tools-tzolov/
[^13]: Anthropic, "Manage tool context," Anthropic API Docs, https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/manage-tool-context
[^14]: Ola Hungerford, Sam Morrow, Luca Chang, "Tool Annotations as Risk Vocabulary: What Hints Can and Can't Do," MCP Blog, March 16 2026, https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
[^15]: Simon Willison, "The lethal trifecta for AI agents," June 16 2025, https://simonwillison.net/2025/jun/16/the-lethal-trifecta/
