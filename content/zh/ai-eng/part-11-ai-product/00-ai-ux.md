---
title: "AI 产品的用户体验（UX）"
description: "流式输出是信任信号，不是传输细节。三种故障模式（空白屏幕、中途卡住、无法核实的论断），以及各自该怎么做。"
slug: 00-ai-ux
part: 11
chapter: 0
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/01-streaming
  - part-2-working-with-model-apis/00-calling-models
  - part-6-retrieval-augmented-generation/05-citations-and-grounding
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# AI 产品的用户体验（UX）

有三件事会摧毁用户对 AI 产品的信任，而它们都不是你盯着 API 仪表盘能猜到的。第一个 token 到达之前，屏幕空白了三秒。答案写到一半，光标卡住两秒，用户以为系统崩了。模型笃定地说出一个事实，用户却无从核实。三者的成因不同，解法也不同，而且没有一件会在日志里报错。

本章讲的是流式（streaming）、不确定性（uncertainty）和引用之上的 UX 层。Server-Sent Events 的机制、流式请求的生命周期、引用的 API 形态，都在[流式输出](../part-2-working-with-model-apis/01-streaming.md)和[引用与事实依据](../part-6-retrieval-augmented-generation/05-citations-and-grounding.md)里。这里问的是那两章没回答的问题：字节开始流动之后，用户实际需要看到什么，为什么？

## 流式输出是信任信号，不是传输细节

总延迟相同时，流式响应让人感觉比缓冲式响应快大约 40%。[^1] 这个效果几乎全部来自首 token 时间（time-to-first-token，TTFT）。300 毫秒就出现第一个字符，用户就知道系统活着、在干活、归他所有；同样 300 毫秒的空白屏幕，什么也没告诉他。一旦 token 开始往外冒，用户已经在读了。全部收益就在这里。

接下来这个数字让大多数团队不舒服：一个普通英语读者的阅读速度大约是每秒 4.8 个 token。[^2] 当今前沿 API 的流式速度是每秒 80 到 100 个 token，Groq 级别的硬件能到 480 至 525。[^3] 第一个 token 渲染出来之后，后续每个 token 的到达速度都超过用户的阅读速度。以 10 倍于阅读速度生成 token，对用户体验的增益是零。从第二个 token 起，真正要紧的是到达节奏稳定。杀死体验的是卡顿，不是吞吐量。

这就把大多数团队搭的仪表盘废掉了。平均每输出 token 时间（time-per-output-token，TPOT）是个标量，它把一整条流平均成一个数。一条顺畅的流和一条中间冻住两秒的流，可以算出完全相同的 TPOT。用户说其中一条坏了，仪表盘说两条都是绿的。正确的指标是按每条流统计的 token 间隔延迟（inter-token latency）直方图，告警看的是每条流内部最坏间隔的 P99，而不是跨流的平均值。[^4]

![一条横向时间轴，上面有四条并行的时间线。每条线都画了理想的分词下发轨迹（平滑斜线）和实际下发轨迹，但偏离方式各不相同：情形 A 与理想完全重合，情形 B 起步晚了然后平行推进，情形 C 全程斜率只有一半，情形 D 先与理想重合，接着两秒内走平，然后恢复](../../../ai-eng/part-11-ai-product/images/00-streaming-failure-modes.png)
*四条流用户都觉得“慢”，但需要四种不同的修法；情形 D 中间那段停顿，正是 TPOT 仪表盘看不见的故障。*

三个数字可以作为 SLO 讨论的锚点。交互式聊天：P95 的 TTFT 低于 500 毫秒。IDE 补全：P95 低于 100 毫秒。长文生成：最多 3 秒可以接受，前提是后续的流保持平滑。[^1] 这些是用户感知的阈值，不是基础设施的愿景。

有一种故障模式值得单独点名，因为它无处不在：号称流式，其实不流。NGINX 的 `proxy_buffering on` 是默认值，它把 token 攒在代理上，直到缓冲区填满或者响应结束，然后一次性冲出去。在你的服务器上，字节确实是端到端流式传输的；用户看到的是一片空白，然后整段回答瞬间到达。解法是 `proxy_buffering off`、`gzip off`、`proxy_read_timeout 86400s`，再加响应头 `X-Accel-Buffering: no`。每 15 至 30 秒发一次心跳事件，可以避免长生成过程中代理超时。走 CDN 的链路通常还需要一条显式的流式透传规则。[^1]

## 什么时候假流式才是正确答案

真流式的麻烦在于：你在校验完整输出之前就开始吐 token 了。如果响应必须先通过 schema 校验、内容审核或安全检查才能展示，真流式就逼你做一个艰难的选择：先把未校验的内容发出去，校验失败再撤回；或者把完整响应缓住，在客户端逐字播放动画。后一种就是假流式（fake streaming），对短响应来说它几乎总是正确的选择。

判断规则很简单。如果校验完的输出能在 1 至 2 秒内落地，就用假流式。用户看到的打字动画和真流式感觉一模一样，而你在第一个字符出现之前就已经校验完了全文。超过 2 秒，缓冲等待本身就变成 UX 问题，这时你需要真流式，外加一条校验失败时回滚的路径。

另一种让流式模式彻底失效的场景是推理模型。开启扩展思考（extended thinking）的 Claude Opus 4.7，TTFT 的 P50 是 28 秒、P95 是 67 秒。[^3] 一个空白闪烁 67 秒的打字光标，用户只会读成崩溃。目前正在成形的做法是——截至 2026 年中还没有行业共识——把光标换成结构化状态提示（“正在思考，可能需要一分钟”），并且在 API 暴露推理 token 时，把它们放进可折叠面板里显示，让等待显得有产出。最接近参考实现的是 Claude.ai 客户端。

## 传达不确定性而不摧毁信任

模型输出的语气总是笃定的，而它正确的概率无法从文字表面读出来。这是结构性问题，不是 bug。产品层面的问题是：UI 该显示什么。

大多数团队的第一反应是给一个数值化的置信度分数。新加坡国立大学 2024 年 11 月的一项研究（N=252）发现，校准失当的置信度比完全不显示置信度更糟。[^5] 过度自信的 AI 导致误用（misuse）：用户听信了他们自己本来能识破的错误建议。自信不足的 AI 导致弃用（disuse）：用户拒绝了正确的建议。只有 26% 至 29% 的用户能自己察觉校准失当；而在告知用户系统校准失当（“这个系统倾向于过度自信”）之后，误用减少了，弃用却增加了，整体任务准确率没有变化。结论很直白：除非你有一条能产出校准估计的流水线，否则不要显示数值置信度。大多数团队没有。

模型自报的置信度是所有信号里最弱的。截至 2026 年 6 月，在六个开放权重模型和三个基准上，模型给自己回答打的置信度比给别人回答打的高出最多 26%。[^6] 问模型“你对这个答案有多自信”会得到一个数字，而这个数字并不预测正确性。它反映的是模型，不是模型的输出。

真正有效、而且效果出奇好的做法，是让模型用自己的话把不确定性说出来。Kim 等人 2024 年发表在 FAccT 的论文（N=404，预注册）发现，第一人称的模糊表述（“我不太确定，但……”）显著降低了被试认同错误 AI 答案的倾向，并提升了任务准确率。第三人称的表述（“目前尚不清楚，但……”）趋势相同，但未达到显著性。[^7] 措辞很关键：同样的不确定性内容换一种口吻说出来，效果就不一样。

另一项有实证支持的干预是按论断做可视化的事实性标注。IBM Research 的研究（N=208，2025）比较了六种展示设计，发现按短语整体高亮的着色方式最受欢迎、最受信任，也最容易验证。[^8] 作者明确点出的前提是：只有当你的事实性打分器可靠时，这招才有效。给错误文字贴一个绿色徽章，比不贴更糟，因为用户此时会把注意力锚在徽章上，而不是自己的判断上。

不依赖任何花哨基础设施、又能稳稳见效的默认做法是：把第一人称的模糊表述写进提示词，而把数值置信度和颜色标注留给那些你能证明底层信号已校准的场景。模糊表述零成本且有效；标注是实打实的工程投入，赶工做反而会反噬。

## 引用是导航手段，不是信任徽章

引用幻觉（citation hallucination）不再是理论上的担忧。2025 年，大约每 277 篇同行评议论文里就有 1 篇至少含有一条 AI 编造的参考文献，而 2023 年这个比例是 2828 分之一。[^9] 斯坦福的研究者在主流 AI 法律检索工具上记录到超过 17% 的幻觉率。[^10] 不可核实的 AI 论断，其风险已经从“用户可能被误导”变成“用户已经照着模型编造的谎话行动了”。

引用在 UX 层解决核实问题。把论断链接到支持它的那段原文的行内引用，会把用户与输出的关系从“我必须信它”变成“我两秒钟就能查”。Perplexity 的产品身份就建立在这个能力上：每次查询访问约 10 个页面，每条响应里给出 3 至 4 个带引用的来源，每一个都可点击，点开就是原文。[^11] 设计原则是：引用是导航手段，不是质量印章。用户负责抽查，你负责让抽查变得毫不费力。

机制之所以重要，是因为引用并不都一样。基于提示词的引用，也就是让模型在写作时自己插入 `[source 3]` 标记，会产出捏造的指针：一段看起来很像样的字符区间，指向空无。模型生成指针的方式和它生成其它一切的方式相同，也就意味着同一套幻觉机制照样生效。Anthropic Citations API 在 API 层面堵住了这个漏洞。模型生成带标注的论断；API 按模型返回的索引，从真实文档里把被引的文本抽出来。模型无法编造出一条指向文档中并不存在的文字的引用。[^12]

```python
import anthropic

def query_with_citations(doc_text: str, question: str) -> list[dict]:
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {"type": "text", "media_type": "text/plain", "data": doc_text},
                    "citations": {"enabled": True},
                },
                {"type": "text", "text": question},
            ],
        }],
    )
    results = []
    for block in response.content:
        if block.type == "text" and getattr(block, "citations", None):
            for c in block.citations:
                results.append({
                    "claim": block.text,
                    "cited_text": c.cited_text,
                    "start": c.start_char_index,
                    "end": c.end_char_index,
                })
    return results
```

这个 API 有两点对产品决策有用。`cited_text` 字段不计入输出 token，所以相对普通散文生成，加引用在结构上是零成本的。另外截至 2026 年 6 月，Citations API 与结构化输出在同一次调用里不兼容：只能二选一。对于使用没有同等能力的供应商的 RAG 流水线，可用于生产的替代方案是两遍生成（先写答案，再让模型为它标注来源指针），随后做基于 NLI 的校验，确认每条引用确实蕴含它所支持的论断。[^13] 代价是每条响应大约多一次 LLM 调用，换来的是模型无法编造指针的保证。

## 这对你正在构建的界面意味着什么

开篇那三种故障，正好对应三个产品决策。空白屏幕：默认走流式；校验要求时改成假流式；凡是 TTFT 超过五秒的模型，把光标换成结构化状态提示。中途卡住：把 token 间隔延迟按每条流做成直方图埋点，告警盯流内 P99 间隔而不是聚合 TPOT，并且把代理链路配成真的会把 token 透传出去。无法核实的论断：对所有容易出不确定性的回答，把第一人称模糊表述写进系统提示词；有可用的事实依据文档时使用 API 级引用；在能证明校准之前不要给数值置信度。三件事彼此独立，也各自以各自的方式失败。三件都做对的产品，才是用户会回来的那个。

## 参考文献

[^1]: Tian Pan, "Streaming AI Applications in Production: What Nobody Warns You About", April 2026. https://tianpan.co/blog/2025-11-07-streaming-ai-applications-production

[^2]: Liu, Jiachen et al., "Andes: Defining and Enhancing Quality-of-Experience in LLM-Based Text Streaming Services", arXiv:2404.16283v2, December 2024. https://arxiv.org/html/2404.16283v2

[^3]: Digital Applied, "AI Model Latency Benchmarks 2026: TTFT and Throughput", April 2026 (10,000 probes per provider pairing). https://www.digitalapplied.com/blog/ai-model-latency-benchmarks-2026-ttft-throughput

[^4]: Tian Pan, "Inter-Token Jitter: The Streaming UX Failure Your p95 Dashboards Can't See", April 27, 2026. https://tianpan.co/blog/2026-04-27-inter-token-jitter-streaming-ux-blind-spot

[^5]: Li, Jingshu et al., "Overconfident and Unconfident AI Hinder Human-AI Collaboration", National University of Singapore, arXiv:2402.07632v2, 2024. https://arxiv.org/html/2402.07632v2

[^6]: Xiong et al., "Large Language Models Are Overconfident in Their Own Responses", arXiv:2606.03437, June 2026. https://huggingface.co/papers/2606.03437

[^7]: Kim, Sunnie S. Y. et al., "I'm Not Sure, But...: Examining the Impact of Large Language Models' Uncertainty Expression on User Reliance and Trust", FAccT 2024, arXiv:2405.00623. https://arxiv.org/abs/2405.00623

[^8]: Do, Hyo Jin et al. (IBM Research), "Highlight All the Phrases: Enhancing LLM Transparency through Visual Factuality Indicators", arXiv:2508.06846, 2025. https://arxiv.org/html/2508.06846

[^9]: Columbia University School of Nursing news, "AI Blamed For Rise In Fabricated Citations Found In Recent Research Papers", May 2026. https://www.nursing.columbia.edu/news/ai-blamed-rise-fabricated-citations-found-recent-research-papers

[^10]: Forbes Business Council, "What Business Leaders Need To Know When AI Fabricates Legal Citations", May 2026. https://www.forbes.com/councils/forbesbusinesscouncil/2026/05/27/what-business-leaders-need-to-know-when-ai-fabricates-legal-citations/

[^11]: Stackmatix, "The Multi-Source Citation Strategy for AI Search", 2025 (Perplexity citation behavior as of 2024). https://www.stackmatix.com/blog/perplexity-optimization-multi-source-citation

[^12]: Anthropic, "Citations", API Documentation, June 2026. https://docs.anthropic.com/en/docs/build-with-claude/citations

[^13]: Medium / Data Science Collective, "Anthropic-Style Citations with Any LLM", March 2025. https://medium.com/data-science-collective/anthropic-style-citations-with-any-llm-2c061671ddd5
