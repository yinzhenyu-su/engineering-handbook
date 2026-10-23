---
title: "压缩与上下文预算"
description: "上下文更多不等于更好。中间遗忘（lost-in-the-middle）与上下文腐化（context rot）如何拖累长提示，以及何时该裁剪、摘要还是重新检索。"
slug: 02-compression-and-context-budgets
part: 5
chapter: 2
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - 00-context-is-more-than-the-prompt
  - 01-context-assembly-and-routing
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 压缩与上下文预算

一个窗口有一百万 token 的模型，塞 10 万 token 进去时的回答，可能比塞 1 万 token 更差。

这不是某个模型独有的怪癖。2025 年 7 月，Chroma 测了 18 个前沿模型，包括 GPT-4.1、Claude Opus 4、Gemini 2.5 Pro 和 Qwen3-235B。输入越长，它们无一例外都变差，哪怕任务简单到只是复述一串词。[^1] 在 LongMemEval 上，把同一段对话压成 300 token 的聚焦提示，每个 Claude 模型的得分都高于塞满 113,000 token 的版本。[^1]

窗口更大不等于"记住得更多"，只是绳子更长，更容易把自己吊上去。等提示词和检索都跑通了，上下文工程（context engineering）剩下的工作就是刻意把窗口留小，并且清楚什么时候该用哪种压缩动作。

## 长上下文失效的两种方式

第一种失效是位置性的。Liu 等人（TACL 2024）在 GPT-3.5-Turbo、Claude、MPT 和 LongChat 上做多文档问答，把含答案的那份文档依次放到 20 份文档上下文的每一个位置上。[^2] 准确率画出 U 型：答案在第 1 位或第 20 位时高，在中间时低。最差的位置上，GPT-3.5-Turbo 掉到 **52.9%，低于它 56.1% 的闭卷成绩。** 对模型来说，一份文档都不给，反而比把正确文档埋在中间更好。[^2]

![多文档问答的 U 型准确率曲线。第 1 位与第 20 位准确率高，中间下降，最低点低于闭卷基线。](../../../ai-eng/part-5-context-engineering/images/02-lost-in-the-middle.png)
*检索 20 份文档时，模型对开头和结尾注意力最强；答案落在中间，效果比不给文档还差。*

第二种失效来自长度本身。Chroma 把它命名为**上下文腐化（context rot）**：随输入长度增长的、持续且可测量的性能退化，与"你没地方了"这种二值失败是两回事。[^1] 三个因素叠加。注意力稀释：softmax 之下，每个相关 token 的权重与 1/N 成正比，输入翻倍，其中每条事实的信号强度就减半。干扰项压制：哪怕只有一段话题相关但内容错误的段落也会拉低准确率，四段干扰项的效果还是非线性叠加的。最后再叠上中间遗忘偏差。

Chroma 有个发现确实奇怪。模型在打乱的干草堆上得分，*高于*内容相同但逻辑连贯的文章。[^1] 连贯的行文会造出更像样的干扰项，模型的注意力摊得更平，针被稀释了。连贯性——你的检索流水线拼命想产出的那个东西——反而可能害你。

## 压缩之前先做两件便宜的补救

如果这一章你只做两件事，今天就先把它们做进检索流水线里。

**把检索文档数卡在 20 份左右。** Liu 等人测到，超过 20 份之后检索器召回率还在涨，阅读器的准确率却已经平了：第 21 份到第 N 份大约只贡献 **GPT-3.5-Turbo 上 1.5%、Claude 上 1% 的准确率**，代价是满额的延迟和满额的 token 成本。[^2] 过了 20 份，瓶颈不再是你的重排器，而是模型的注意力。用好的重排器从 top-50 收到 top-10，胜过用顶级重排器从 top-10 放到 top-50。

**第 1 名放最前，第 2 名放最后。** U 型曲线是一条免费的决策规则。得分最高的文档放第 0 位，第二高的放上下文末尾，其余的塞中间——反正那里的注意力最稀薄：

```python
def order_for_u_curve(docs: list[dict]) -> list[dict]:
    """Place top-ranked at start, second at end, rest in middle.
    Mitigates lost-in-the-middle (Liu et al., TACL 2024)."""
    ranked = sorted(docs, key=lambda d: d["score"], reverse=True)
    if len(ranked) <= 2:
        return ranked
    head, tail, middle = ranked[0], ranked[1], ranked[2:]
    return [head, *middle, tail]
```

有一个前提要说清。Chroma 的 NIAH 变体在 11 个针位置上都复现不出明显的 U 型曲线。[^1] 他们的干草堆没有按相关性排过序，这大概压掉了首位优势。到 2026 年年中，这个分歧仍未解决；不过照 Liu 的建议做，成本是零，那就做。

## 裁剪、摘要、重新检索：按情况选一个

上下文一旦大到必须切，你手上只有三个动作，而且它们不能互换。判断依据是：缺失的信息长什么样，以及你能不能把它拿回来。

- **裁剪（trim）**丢掉最早的 N 个 token 或对话轮次。它免费、即时，也不分青红皂白。适用场景是上下文本来就按时序排列（比如只关心最近几轮的聊天），被丢掉的东西是死重量，不是状态。
- **摘要（summarize）**用一次 LLM 调用生成的摘要替换长历史，摘要里写决策、约束和未完成的任务。它多花一次模型调用，而且设计上就是有损的。适用于有状态的长跑任务、编码智能体、研究智能体、多轮解题——这些场景里，早先的*决策*仍然约束着后面的轮次，尽管原文不必逐字保留。
- **重新检索（re-retrieve）**为过时的数据丢掉当前上下文，重跑一次检索查询。只要缺失的信息能从索引里查出来，就用它。面对稳定语料做事实查询时，重新检索优于摘要，因为它是无损的：你付出检索成本，换回逐字的原始材料。

决策树很短：

```text
信息还在某个索引里吗？      -> 重新检索
更早的状态现在还有约束力吗？  -> 摘要
以上都不是                  -> 裁剪
```

一个重构做到一半的编码智能体需要摘要记忆："我们定了用 Postgres，schema 在 `db/schema.py`，还有三个测试没过。" 一个回答"我们的退款政策是什么"的客服机器人需要针对当前政策文档重新检索。一场闲聊里用户刚追问了一句，那就把最近几轮之前的内容裁掉。

把三者搞混，既烧钱又掉质量。裁剪编码智能体会丢掉 schema 决策，智能体自己再推一遍，推错。给事实查询做摘要，逐字引用会变成转述，再变成幻觉。对多轮谈判做重新检索，会丢用户在三个轮次之前确立的约束。

Ma 等人的 SELF-ROUTE（2024）把这件事看成按查询路由的问题，而不是一条固定流水线。[^3] 你不必为整个产品只选一种策略。同一个智能体可以在闲聊时裁剪、在会话边界做摘要、在工具返回"结果可能已经变化"时重新检索。这才是生产形态，而不是"我们是一家 RAG 公司"或者"我们是一家长上下文公司"。

## 摘要在生产环境里实际怎么跑

最小形态是一个带摘要记忆的滚动缓冲。最近几轮原样留着。缓冲一旦越过阈值，就把更早的内容摘要成一个块，追加到滚动摘要上，丢掉原始轮次：

```python
from dataclasses import dataclass, field

@dataclass
class CompactionState:
    summary: str = ""
    recent: list[dict] = field(default_factory=list)
    tokens: int = 0

def maybe_compact(state, new_turn, summarize_fn,
                  budget=8000, keep_turns=4):
    state.recent.append(new_turn)
    state.tokens += len(new_turn["content"]) // 4  # 粗略估算

    if state.tokens <= budget:
        return state

    older = state.recent[:-keep_turns]
    state.recent = state.recent[-keep_turns:]
    new_summary = summarize_fn(state.summary, older)
    state.summary = new_summary
    state.tokens = (len(new_summary) // 4
                    + sum(len(t["content"]) // 4 for t in state.recent))
    return state
```

`summarize_fn` 是承重的那部分，而"总结一下这段对话"是错误的提示词。通用摘要优化的是流畅度，会悄悄丢掉你真正需要的东西：生效中的约束（"绝不存 PII"）、具体取值（"用户账号 ID 是 88421"）、尚未处理的工具结果、已经做出的决策。把摘要指令写成结构化抽取，而不是复述。把类别列出来：*已做出的决策、生效中的约束、未完成的任务、关键取值。* 测试办法是在压缩触发之后问一个只需要压缩前才确立的事实的探测问题。如果答案变了，说明摘要把它弄丢了。

Anthropic 在 2026 年初把服务端压缩作为 beta 功能上线（header：`compact-2026-01-12`）。输入超过阈值时自动触发（默认 150,000 token，最低 50,000），用同一个模型跑一次内部摘要调用，并在被丢弃的消息位置返回一个 `compaction` 块。[^4] 它附带两个计费陷阱。摘要调用按你请求的那个模型的全额输入/输出价计费，没有换用更便宜摘要器的选项。而顶层 `usage.input_tokens` 字段不含压缩那一轮；要拿到真实成本，得把 `usage.iterations` 一路累加。忽略这个字段的成本追踪器，每次压缩事件都会悄悄少报几万 token。[^4]

> [!WARNING]
> 工具调用结果是压缩的隐形杀手。长跑的智能体会积累大块 `tool_result`；摘要器把它们揉成"我调了五次搜索工具，拿到了关于 X 的结果"。下一轮智能体看不到实际数据，只好重新调工具，把刚刚花过钱算出来的东西再算一遍。Anthropic 的文档建议用 `tool_result_clearing` 丢掉已处理的工具输出，或者用 `pause_after_compaction`，在对话继续之前原样保住最近的工具结果。[^4] 两者都不做，压缩会把工具调用成本大约翻倍，而不是把它降下来。

## 上下文隔离：比压缩更管用的那一步

有时候正确答案不是压缩一段长上下文，而是根本别把它拼出来。

Anthropic 的多智能体研究系统用一个 Opus 4 主智能体，把子任务分派给 Sonnet 4 子智能体。每个子智能体在自己的隔离上下文窗口里干活，返回大约 50 到 200 token 的浓缩结果。主智能体永远看不到搜索轨迹、死路和工具往返。Anthropic 报告这套架构在研究任务上比单个 Opus 4 智能体好 90.2%，代价是分摊在各隔离窗口里的总 token 消耗约为 15 倍。[^5]

这是对上下文腐化的架构级回答。子智能体用委派代替摘要：主智能体读的不是 10 万 token 的搜索历史再加一份摘要，而是一个 200 token 的结果。代价是总 token 更多；好处是没有任何单个窗口长到会腐化。当你发现自己在为某个智能体循环搭一套繁复的压缩逻辑时，先问一句：这个循环是不是该改成一次子智能体调用。这个模式在架构尺度的样子——它在系统图里和队列、有状态服务并排的位置——见《High-Level Design Handbook》的[智能体系统一章](../../hld/part-9-ai-ml-system-design/03-ai-agent-architectures.md)。

## 有意识地把预算花出去

统一的习惯是给每一层上下文一个明确的 token 配额，并在每次模型调用之前强制执行。系统提示、对话历史、检索文档、工具定义、输出预留。输入超预算时，在 API 拒绝这次调用*之前*就压缩，而不是之后：

```python
def allocate_budget(system_t, history_t, retrieved_t,
                    max_context=128_000, output_reserve=4_096):
    available = max_context - output_reserve
    system = min(system_t, available)
    rest = available - system
    history = min(history_t, int(rest * 0.40))
    retrieved = min(retrieved_t, rest - history)
    overflow = (system_t > system or history_t > history
                or retrieved_t > retrieved)
    return {"system": system, "history": history,
            "retrieved": retrieved, "overflow": overflow}
```

这些百分比是起点，不是定律。用你在[你的第一套评测集](../part-4-eval-fundamentals/02-your-first-eval-set.md)里建好的评测集去调：变动历史与检索的配比，测答案质量，找拐点。多数团队会发现，他们把窗口的 60% 花在了毫无贡献的对话历史上，只有 10% 花在真正干活的检索文档上。重新分配之后，同一个模型、同样的提示词，答案就开始变好。

这一章要带进[记忆与状态](./03-memory-and-state.md)和[工具上下文与元数据](./04-tool-context-and-metadata.md)的一句话是：上下文窗口是一笔要花出去的预算，不是一个要填满的袋子。把它填满，模型会惩罚你。

## 参考文献

[^1]: Hong, Kelly; Troynikov, Anton; Huber, Jeff. "Context Rot: How Increasing Input Tokens Impacts LLM Performance." Chroma Technical Report, July 14, 2025. https://research.trychroma.com/context-rot
[^2]: Liu, Nelson F.; Lin, Kevin; Hewitt, John; Paranjape, Ashwin; Bevilacqua, Michele; Petroni, Fabio; Liang, Percy. "Lost in the Middle: How Language Models Use Long Contexts." TACL, Vol. 12, pp. 157-173, 2024. arXiv:2307.03172. https://arxiv.org/abs/2307.03172
[^3]: Ma, Shuai; et al. "Retrieval Augmented Generation or Long-Context LLMs? A Comprehensive Study and Hybrid Approach." arXiv:2407.16833, 2024. https://arxiv.org/abs/2407.16833
[^4]: Anthropic. "Compaction." Claude API Documentation (beta, `compact-2026-01-12`). Accessed June 2026. https://docs.anthropic.com/en/docs/build-with-claude/compaction
[^5]: Anthropic. "How we built our multi-agent research system." Anthropic Engineering Blog, 2025. https://www.anthropic.com/engineering/multi-agent-research-system
