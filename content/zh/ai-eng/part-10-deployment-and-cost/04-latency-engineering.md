---
title: "延迟工程"
description: "为什么 LLM 延迟不是一个单一数字：五个阶段的瀑布分解、TTFT 与总耗时的区别、并行工具调用、前缀缓存，以及在 UX 侧弥合差距的感知延迟技巧。"
slug: 04-latency-engineering
part: 10
chapter: 4
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/01-streaming
  - part-2-working-with-model-apis/07-prompt-caching
  - part-7-agent-engineering/02-the-agent-loop
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 延迟工程

用户在你的聊天产品里敲下一个问题。三秒白屏。他重试了一次。然后走了。

直觉会把这件事归成“模型太慢”，然后伸手去找一个更快的模型。这个判断有一半的时候是错的。LLM 延迟不是一个单一数字，它是一条由五个不同阶段叠加而成的瀑布，每个阶段的瓶颈不同、修法不同，与用户真实感受的关系也不同。

本章讲的是怎么拆开这条瀑布、选对指标（它不是你以为的那个），然后把优化打在正确的阶段上。

## 瀑布分解

每个 LLM 请求都能拆成五个相加的阶段：[^1]

```text
TTFT = T_network + T_queue + T_tokenization + T_prefill
E2E  = TTFT + (TPOT x output_tokens)
```

- **网络入口（T_network）：** 从客户端到推理服务器的传输。区域内 5-30 ms，跨洲 80-220 ms。
- **排队等待（T_queue）：** 等一个 GPU 批处理槽位的时间。低负载时接近零，高并发时占主导。
- **分词（T_tokenization）：** CPU 侧把文本转成 token ID。1K token 约 10 ms。
- **预填充（T_prefill）：** GPU 对所有输入 token 做一次前向计算。受算力约束。走 Azure PTU 的 GPT-4 Turbo 上，每个输入 token 大约给 TTFT P95 加上 0.24 ms。[^2]
- **解码（TPOT）：** 受内存带宽约束。每个输出 token 一次前向计算。决定流式吞吐。

![一张横向堆叠条形图：一个 LLM 请求从左到右被拆成五段不同颜色的区间（网络、排队、分词、预填充、解码），前四段合并为 TTFT；下方第二条柱是同一个请求在开启前缀缓存、预填充几乎被消掉之后的样子。](../../../ai-eng/part-10-deployment-and-cost/images/04-latency-waterfall.png)
*同一个请求，开不开前缀缓存。只要提示词不是几句话就完，预填充就在 TTFT 里占大头——这就是为什么缓存是杠杆最大的那一个动作。*

在 H100 SXM5 上、1,024 token 提示词、400 ms 的 TTFT P99 预算下：网络大约 20 ms，排队 50 ms，分词 10 ms，预填充 280 ms。[^1] 预填充占大头。所以提示词长度和前缀缓存是 TTFT 最大的两个杠杆。

排队是那个藏起来的变量。在 M/M/1 动态下，P99 排队等待大致按 1/(1 - 利用率) 放大。利用率 80% 时，P99 长尾延迟比 20% 时差 5 倍。那条能用的 SLO 规则背后的数学就是这个：60-70% 利用率就触发扩容，绝不等到 80%。[^1]

## 真正重要的指标是 TTFT

如果这章你只带走一件事，就是这句：每秒 token 数（tokens-per-second）不是面向用户的指标，TTFT 才是。

推理从 1968 年 NNGroup 的三档极限模型（Miller）开始，1991 年由 Card 等人做了细化。三个人类感知阈值，五十多年没变：[^3]

- **100 ms 以内。** 感觉是瞬时的。算直接操作。
- **100 ms 到 1 秒。** 用户察觉到延迟，但心流没断。
- **1 到 10 秒。** 用户得主动等待。必须有进度指示。
- **超过 10 秒。** 注意力开始游走。用户会离开。

套到 LLM 聊天上：TTFT 低于 500 ms 感觉灵敏，500 ms 到 1 秒可以接受，1 到 2 秒就已经被记成“慢”，超过 2 秒直接流失。[^4]

为什么单靠流式（streaming）救不了慢 TTFT：流式掩盖的是第一个 token 之后的延迟。一个 TTFT 2.5 秒、之后流得很欢的系统，先给用户 2.5 秒白屏，然后才开始出字。哪怕端到端总时长更长，用户也会觉得它比“TTFT 400 ms、后面流得慢一点”的系统更慢。[^4]

推理模型把这套东西整个打破。Claude Opus 4.7 在最大思考预算下的扩展思考，TTFT P50 是 28 秒；GPT-5.5 Pro 在高推理强度下到 67 秒。[^5] 对交互式 UX 来说，推理模式不是一个更慢的变体，而是另一个延迟量级，应该按后台异步任务来设计，不是当成聊天回复。

第二个陷阱：别盯均值。

> [!WARNING]
> **P50、P95、P99 要分开看。200 ms 的 P50 完全能和 3 秒的 P99 并存**，这意味着有 1% 的用户等了均值暗示时长的 15 倍。P99/P50 的比值在各家供应商上平均 2.1 倍，最差的配对能到 3.2 倍。[^5] 容量要按预期峰值并发下的 P99 来定，不是按平均负载下的 P50。

另一个常见错误：只在一个并发用户下压测。那个数字不含排队，也不含批处理干扰，拿来定容毫无用处。永远在预期的 P95 并发请求数下测。

## 毫秒到底花在哪

按 UX 类别给的默认 TTFT P99 目标，截至 2026 年中：[^1]

| 用户体验类别 | TTFT P99 预算 | ITL P99 预算 |
|---|---|---|
| 实时语音 | 150 ms | 30 ms |
| 行内代码补全 | 100 ms | 25 ms |
| 聊天 / 交互 | 300 ms | 50 ms |
| RAG 增强聊天 | 400 ms | 80 ms |
| 后台智能体 | 3,000 ms | 200 ms |

这些目标对选模型、选供应商的影响，比模型能力本身更大。300 ms 以下的 TTFT，现实中只有 Groq LPU（Llama 4 405B 上 P50 约 0.18 s）或 Cerebras 晶圆级芯片（Llama 4 70B 上 P50 约 0.16 s）做得到。[^5] 前沿闭源模型（Claude Opus 4.7 标准版、GPT-5.5 标准版、Gemini 3 Pro）P50 在 0.85-1.12 s，聊天够用，补全不行。Cursor 的 IDE 补全之所以专门路由到 Groq，就是因为 100 ms 的预算把其他选项全排除了。

决策规则：每个 UX 类别单独选模型和供应商。同一个产品，行内补全走 Groq，面板聊天走 Claude Opus，因为它们的延迟预算不是一回事。

## 前缀缓存是 TTFT 上杠杆最大的单一手段

只要提示词超过几百个 token，预填充就在 TTFT 里占主导。砍预填充是最大的一块实际收益，而砍预填充的办法，就是跳过那些已经算过的 token。

Glean 公布过测量结果：走 Azure PTU 的 GPT-4 Turbo 上，每个命中的输入 token 大约省下 0.15 ms 预填充。[^6] 缓存 1,000 个 token 大约省 100 ms TTFT。Glean 的生产系统在缓存温热的请求上，TTFT 从 4.3 秒降到 0.6 秒，模型一个字没换，提升 7 倍。[^7]

要用上前缀缓存，得把提示词排成：公共内容（系统提示词、工具 schema、少样本示例、文档上下文）在前面，每请求变化的内容（用户 ID、时间戳、当前问题）在最后。缓存键是到第一个不同 token 为止的前缀哈希。前面 token 有一个字节的差异，整个下游缓存全部失效。

```text
[system prompt] -> [tools] -> [few-shot examples] -> [document] -> [user query]
        cacheable                               variable
```

陷阱：开了前缀缓存却不测命中率。团队把功能发上去，没看到改善，也不知道为什么。修法和上一章那个缓存杠杆一样：打点 `cache_read_input_tokens`，盯着它变成非零。

各家供应商缓存实现的细节，见[提示词缓存](../part-2-working-with-model-apis/07-prompt-caching.md)。

## 并行工具调用

在智能体流水线里，工具执行占总智能体延迟的 35-60%。[^8] 当模型在一个回复轮里吐出多个 `tool_use` 块，正确的做法是并发派发，并在下一步推理之前，把所有结果放进一条 user 消息里返回。

算术很直白：三个独立的 200 ms 工具调用，串行 600 ms，并行 200 ms。LLM-Tool Compiler 论文在一个 Copilot 级平台上测到，并行调用数最高达到基线方法的 4 倍，延迟最多降 40%。[^9]

```python
import asyncio

async def dispatch_parallel_tools(tool_uses: list) -> list:
    """Execute all tool_use blocks from one model turn concurrently.
    Wall-clock cost = max(individual latencies), not sum.
    All results returned in a single list for inclusion in ONE user message.
    """
    async def run_one(block):
        result = await TOOL_REGISTRY[block["name"]](**block["input"])
        return {"type": "tool_result", "tool_use_id": block["id"], "content": result}

    return await asyncio.gather(*[run_one(b) for b in tool_uses])
```

Anthropic API 上有一条关键的消息历史规则：一个并行批次里的所有工具结果必须放在同一条 user 消息里返回，不能拆到几条消息里。[^10] 拆开会训练模型在后续轮次里不再发并行调用。

值得知道的失效模式：隐藏耦合。

你把工具调用并行化，就把串行执行曾掩盖的依赖关系暴露了出来。下面三类问题全都表现为答错，而不是抛异常：

- **上下文依赖。** 工具 A 读的那个共享变量，本该由工具 B 先填好。
- **共享状态修改。** 两个工具对同一资源做读-改-写。丢失更新的竞态条件。
- **执行时序依赖。** 工具 B 的前置条件是工具 A 的副作用（同一行数据先建后改）。

并行之前先过一遍幂等性测试：这个工具是否原子、是否幂等、是否没有共享可变状态？三条全过就并行，任一条不过就用串行或显式的 DAG 编排。只读操作（向量检索、网页搜索、数据库 SELECT）几乎总是安全的；写操作需要显式排序。工具层面的幂等模式见[工具设计](../part-7-agent-engineering/03-tool-design.md)。

## 感知延迟

接下来这段不花钱，却能捞回不少体感。

感知延迟（perceived latency）是等待的主观体验，它靠三种心理机制偏离真实时钟时间。[^11] token 一增量地流出来，大脑就把系统记成“在干活”而不是“坏了”。生成还没结束，有用内容就先出现，用户已经开始读、已经开始拿到价值。在客观等待时间相同的情况下，流式还造出一种掌控感，把挫败压下去。

一个 AI 产品的流式版本，会话时长比总生成时间相当的非流式界面高 3.4 倍，D7 留存高 2.1 倍。[^12] 同样的墙钟延迟，不同的交付方式，不同的产品结果。

三个能回本的 UX 手段：

- **骨架屏胜过转圈。** 用户一提交就画出消息气泡的轮廓，不等第一个 token。同样长的等待，骨架屏感觉比转圈快约 20%，因为它先把布局预期立住了，也就消掉了内容凭空冒出来的认知冲击。[^11]
- **带上下文的进度文案胜过千篇一律的“Thinking...”** “正在检索文档”或者“正在分析 45,000 token 的上下文”，比跳动的省略号管用，因为它锚定了用户的预期。知道为什么等，就愿意等更久。
- **token 间抖动比均匀的慢更难受。** TTFT 200 ms 但 ITL 偶尔飙到 200 ms，体感比 TTFT 400 ms 配平滑的 30 ms ITL 更差。ITL P99 要和 TTFT 分开盯。

流式是交互式 UI 的默认做法，但有一个重要例外。流式等于 UI 承诺渲染进行中的输出。如果模型第一段就写出语气笃定但错的结论，用户会在模型自我纠正之前先读到它。对结构化输出（JSON、代码、任何要先过 schema 校验才有用的东西），正确做法是：先流进隐藏缓冲区，校验，再揭示。或者用受约束解码（vLLM guided generation、Outlines），把 schema 烘进生成过程。

SSE 线格式和部分渲染的基础件，见[流式](../part-2-working-with-model-apis/01-streaming.md)。本章管的是感知那一面，那一章管的是协议。

## 合起来看

贯穿整章的模式是同一个：别把延迟当成一个数字，也别把中位数当成指标。用户感受到的是长尾。用户感受到的是 TTFT，不是吞吐。用户把白屏当成系统坏了，而不是系统慢。

优化顺序，按收益从高到低：开启前缀缓存并核实命中率；并行化独立的工具调用并检查耦合；按峰值并发下的 P99 规划容量；任何交互式 UI 都用流式加骨架屏；对延迟要求高的 UX 类别，路由到更快的供应商。

本章没讲的是：改完之后怎么安全地把它们上线。那是[上线变更](05-shipping-changes.md)。

## 参考文献

[^1]: Mitrasish, "LLM Inference SLO Engineering: TTFT, ITL, and P99 Latency Budgets for Production AI", Spheron, May 2026, https://www.spheron.network/blog/llm-inference-slo-ttft-itl-latency-budget-guide-2026/
[^2]: Veraj Paruthi, "How input token count impacts the latency of AI chat tools", Glean engineering blog, July 2024, https://glean.com/blog/glean-input-token-llm-latency
[^3]: Jakob Nielsen, "Response Times: The 3 Important Limits", Nielsen Norman Group, 1993, https://www.nngroup.com/articles/response-times-3-important-limits/
[^4]: Tian Pan, "TTFT Is the Only Latency Metric Your Users Actually Feel", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-16-streaming-ttft-latency-perception
[^5]: Digital Applied Team, "AI Model Latency Benchmarks 2026", April 2026, https://www.digitalapplied.com/blog/ai-model-latency-benchmarks-2026-ttft-throughput
[^6]: Veraj Paruthi, "How KV caches impact time to first token for LLMs", Glean engineering blog, July 2024, https://glean.com/blog/glean-kv-caches-llm-latency
[^7]: Tian Pan, "LLM Latency Decomposition: Why TTFT and Throughput Are Different Problems", TianPan.co, March 2026, https://tianpan.co/blog/2026-03-10-llm-latency-decomposition-ttft-vs-throughput
[^8]: Tian Pan, "Parallel Tool Calls in LLM Agents", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-10-parallel-tool-calls-hidden-coupling
[^9]: Boming Zhang et al., "An LLM-Tool Compiler for Fused Parallel Function Calling", arXiv:2405.17438, 2024, https://arxiv.org/html/2405.17438
[^10]: Anthropic, "Parallel tool use", platform documentation, 2026, https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/parallel-tool-use
[^11]: Tian Pan, "The Latency Perception Gap: Why a 3-Second Stream Feels Faster Than a 1-Second Batch", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-20-latency-perception-gap-ai-interfaces
[^12]: KindaTechnical, "Streaming for Better Perceived Latency", April 2026, https://www.kindatechnical.com/claude-ai/streaming-for-better-perceived-latency.html
