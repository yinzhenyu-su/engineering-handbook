---
title: "护栏"
description: "输入过滤器、输出校验器、内容审核模型，以及那个没人提醒你的流式输出与校验之争。"
slug: 02-guardrails
part: 9
chapter: 2
difficulty: intermediate
prerequisites:
  - 00-prompt-injection
  - 01-agent-security-and-the-lethal-trifecta
  - part-2-working-with-model-apis/01-streaming
tags: [llm, ai-safety]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 护栏

Anthropic 在 2025 年 2 月做过一次公开红队演练：339 名越狱者（jailbreaker），约 3,700 小时投入，七天里 80 万次对话交互，目标全部指向一套由 Constitutional Classifiers 防护的 Claude 3.5 Sonnet 部署。没有这套分类器时，86% 的合成高级越狱（jailbreak）能打穿同一个模型；加上之后，越狱成功率降到 4.4%。线上误拒率上升 0.38%，统计上并不显著。算力开销是 23.7%。[^1]

护栏就是为这个而存在的，而它大致能买到的也就是这些：多数攻击被挡住，多数时候有效，代价可观但还能接受。它不是提示注入（prompt injection）的解法。它是应用层的过滤器，位于模型权重之外，在那些简单攻击碰到真正重要的东西之前把它们拒掉。

## 三种放置位置，一套架构

护栏要么是确定性的代码（正则、schema 校验器），要么是独立的分类器（审核 API、护栏模型），应用在你请求管线的三个位置之一。

- **输入护栏（input rail）。** 跑在任何生成之前的原始用户消息上。拦截或改写，向客户端返回错误。
- **输出护栏（output rail）。** 跑在用户看到之前的模型响应上。同样是放行、拦截、改写三种处置，但时序问题难得多（下面细说）。
- **工具调用护栏（tool-call guard）。** 跑在工具参数落到外部系统之前。这是执行业务策略最便宜的位置：“这个账户不能发起超过 1 万美元的转账”，“这个客服坐席没有工单号就不能退款”。

![横向流程图：用户消息从左侧进入输入护栏框，流入中央的 LLM 圆柱体，再穿过输出护栏框，最后到达右侧的用户；LLM 另有一个向下箭头指进工具护栏框，该框连接一个外部 API 图标；输入与输出护栏用靛蓝色高亮，工具护栏用珊瑚色高亮，标注为杠杆最大的检查点](../../../ai-eng/part-9-security-and-governance/images/02-guard-placement.png)
*护栏落地的三个位置。在 agent 系统里，工具调用护栏是杠杆最大的一层，因为业务策略真正生效的地方就在这里。*

多数聊天应用的最小可用栈，就是一个审核 API 组成的输入护栏加一个输出护栏。完整代码如下：

```python
from openai import OpenAI

def check_and_complete(user_input: str) -> str | None:
    client = OpenAI()

    mod = client.moderations.create(
        model="omni-moderation-latest",
        input=user_input,
    )
    if mod.results[0].flagged:
        return None

    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": user_input}],
    )
    output = resp.choices[0].message.content

    out_mod = client.moderations.create(
        model="omni-moderation-latest",
        input=output,
    )
    if out_mod.results[0].flagged:
        return None
    return output
```

OpenAI 的 `omni-moderation-latest` 在所有用量层级都免费，支持文本与图片，覆盖 13 个类别：harassment、hate、illicit、self-harm、sexual、violence，以及每一类的阈值版和未成年版变体。[^2] 两道护栏任一失败都返回 `None`，让调用方统一展示一句笼统的拒绝，不泄露是哪道护栏触发的——这点很重要，因为攻击者会针对具体护栏去调 payload。

## 按部署位置挑选合适的模型

审核分类器本身有三个正经选项，正好对应三种部署现实。

**OpenAI Moderation（`omni-moderation-latest`）。** 已经在调 OpenAI 的一切场景的默认选择。免费、13 个类别、多模态。2024 年 9 月的升级在 OpenAI 内部评测上把多语言表现比上一代 text-moderation 模型提升了约 42%，低资源语言提升最大：泰卢固语 6.4 倍，孟加拉语 5.6 倍，马拉地语 4.6 倍。[^3] 如果你做的是英文产品、预算中等，第一个该拿的就是它。

**Llama Guard 3-8B。** 开源里的标准选项。Llama-3.1-8B 在 MLCommons 分类体系上微调，外加第 14 个类别用于代码解释器滥用。在 Meta 内部的英文响应分类测试上 F1=0.939，同一评测集上 GPT-4 是 F1=0.805（2024 年 7 月）。[^4] 八种支持语言的多语言 F1 从 0.834（泰语）到 0.943（法语）。还有一个 1B INT4 变体，能在普通 Android 硬件上跑到 30+ tokens/s。[^5] 需要因数据驻留要求做本地部署、明确想要 MLCommons 那套分类体系、或者用量大到审核 API 的调用成本开始疼，就用 Llama Guard。

**Azure AI Content Safety。** 在 Azure 生态里就用它。按类别给出严重级别（safe、low、medium、high），开箱即得分级处置；Prompt Shields 提供显式的越狱检测；Groundedness Detection 预览版能抓住一类幻觉。S0 层支持每 10 秒 1,000 次请求，单次调用输入上限 10,000 字符（截至 2026 年 1 月）。[^6]

没有公开的三方对比基准，所以别拿单个 F1 数字过度加权。真正要紧的是三者大致在同一量级（标准测试集上 F1 在 0.85 到 0.94 之间），而且被攻破的地方也大致相同。

## 什么会漏过去

老实说：比宣传的多。面对每一个商业护栏，都有五类记录充分的绕过手法。

最难看的是**字符注入（character injection）**。Mindgard 与兰卡斯特大学的研究（2025 年 4 月）测出：用 emoji 走私对付 Azure Prompt Shield、Meta Prompt Guard、NeMo Guard Jailbreak Detect，攻击成功率（ASR）为 100%。双向文本 99.23% ASR；零宽字符、同形字、附加符号、全角文本各自落在 44% 到 76% ASR 之间。[^7] 机制很简单：LLM 的分词器是在网页规模的文本上训练出来的，会把这些编码当普通英文解出来；护栏分类器往往是个更小的 BERT 级模型，没在同一个分布上训练过。于是护栏看到的是乱码，模型看到的是一条指令，你就有了绕过。缓解办法是在护栏运行前做一遍 Unicode NFKC 归一化，并剥掉零宽字符。ProtectAI 的 v2 模型专门针对这些编码训练，把字符注入的 ASR 从 77.3% 压到 20.26%。[^7]

另外四类思路类似：低资源语言攻击（尤其是 Llama Guard 支持的那八种之外的语言）、靠同义词替换实现的语义改写（TextFooler 这类工具对护栏模型能到 35-65% ASR）、流式场景下的跨块攻击（下面接着说）、以及梯度优化的对抗后缀。这些都没有已知的通用解法；它们全都指向同一件事：要有纵深防御的姿态，而不是把一个护栏模型当成一堵墙。

还有些类别是护栏模型天然弱的地方。Llama Guard 3 明确把 S5（诽谤）、S8（知识产权）、S13（选举）标为需要“更复杂的系统”，因为判定它们需要当下的世界事实知识。[^4] 依赖上下文的伤害（临床医生问药物剂量、安全研究、含暴力的虚构作品）会在每一个只看表层特征的护栏上抬高误报。

## 流式输出与校验：说出口就收不回的问题

这就是那个没人提前告诉你、直到上线才撞上的矛盾。流式输出在 token 生成的同时就往客户端发，用户能看到响应一点点长出来。而标准的内容分类器需要*完整*的响应才能做判断。这两个要求直接冲突。

你有三个架构选择，选哪个取决于“有害内容哪怕短暂可见”有多糟。

**先缓冲再校验（buffer-then-check）。** 所有 token 压到生成结束，跑护栏，然后放行或拦截。护栏通过之前用户什么都看不到。这等于完全放弃了流式的首 token 延迟优势：2 秒的响应意味着第一个字符要等约 2 秒。在医疗、法律或 CBRN 相关的应用里，有害表层内容的泄露是首要风险，这时候这是正确选择。

**先流式再异步回滚（stream-first with async rollback）。** 立即发 token，并行跑护栏，一旦触发就往流里插一条撤回消息。这是 NVIDIA NeMo Guardrails 在 `stream_first: true` 下提供的选项。他们自己的文档把后果写得很直白：“objectionable text might have already been sent to the user. It's the responsibility of the caller to manage this situation.”[^8] 已经出现在屏幕上的 token 收不回来。有些用户已经读完或者复制走了。

**按块渐进校验（chunk-based progressive validation）。** 把 token 缓冲成固定大小的块（NeMo 默认 200 个 token，块边界处带 50 token 的滑动上下文窗口），逐块校验，安全就吐出去，不安全就中止流。[^8] 对于延迟敏感、内容风险中等的对话应用，这是实用的默认做法。局限在于：故意跨边界切开的有害指令（第 N 块里是“mix compound A”，第 N+1 块里是“with compound B to make X”），只要攻击者够小心，就能从上下文窗口底下溜过去。更小的块（64 到 128 token）能缩小 stream-first 部署的暴露窗口，代价是更多的护栏推理调用。

**预测式护栏（forecasting guards）** 是正在出现的一个更硬的选项：它不等有害 token 真的出现，而是根据已经生成的前缀预测*未来*的 token 是否可能不安全。SB Intuitions 的 StreamGuard 工作（2026 年 4 月）报告 92.6% 的准时干预率（在不安全句子结束之前或结束当下完成拦截），F1=97.5、漏检率 4.9%，在 H100 上决策延迟 9.5 ms。[^9] 这个延迟略低于 8B 模型的单 token 生成延迟，也就是说护栏跟得上生成速度，不会变成瓶颈。

```python
def stream_with_chunk_guard(user_input: str, chunk_size: int = 200):
    buffer: list[str] = []
    token_count = 0

    for delta in generate_stream(user_input):
        buffer.append(delta)
        token_count += 1

        if token_count >= chunk_size:
            text_chunk = "".join(buffer)
            if guard_check(text_chunk) == "unsafe":
                yield None
                return
            yield text_chunk
            buffer = []
            token_count = 0

    if buffer:
        text_chunk = "".join(buffer)
        if guard_check(text_chunk) != "unsafe":
            yield text_chunk
        else:
            yield None
```

调用方处理 `None` 的方式是清掉已渲染的块并显示错误。这就是回滚契约。

## 延迟预算是实打实的约束

一条有用的经验法则：把端到端响应时间的 10% 留给安全工作。2 秒的 p95 聊天体验里，那大约是 200 ms。把内容审核（10-50 ms）、PII 检测（20-80 ms）、schema 校验和一个毒性分类器串行堆起来，光这些就能吃掉 200 到 400 ms。[^10] 一轮 LLM-as-validator 调用（用另一个模型给响应打分）再加 200-800 ms。在 agent 的一堆内部 LLM 调用上串上几道这样的东西，生成还没开始，用户感知的延迟就已经翻倍了。

修法都是显而易见的：独立的护栏并行跑；把快的预过滤（正则黑名单、小的编码器分类器）放在昂贵的护栏模型前面，让便宜的拒绝先短路掉；只给接触不可信输入的 LLM 调用加护栏。可信数据服务之间的内部编排调用不需要内容审核。但多数团队的第一版管线里，它们照样会走一遍。

## 工具调用护栏是杠杆最大的一层

内容审核占了所有话题，但在 agent 系统里最值钱的护栏在工具参数上，不在文本上。校验 `send_email` 的收件人在白名单内、`delete_file` 的路径在工作区内、退款金额低于该档位的上限、SQL 查询里没有未参数化的用户输入——这些都是针对业务策略的确定性检查，每毫秒挡下的损害都远超任何文本分类器。

[致命三要素](./01-agent-security-and-the-lethal-trifecta.md) 里那个出口白名单，真正落地的位置就是这一层。你大部分可审计的策略也在这层。说“这条消息看起来不安全”的内容分类器是概率性的；说“这个账户没有 500 美元以上的退款权限”的工具护栏是确定性规则，好测、好记日志、在合规评审里好辩护。

## 实际该部署什么

典型聊天产品第一天上线的配置：OpenAI Moderation 做输入护栏，OpenAI Moderation 做输出护栏，输入在进护栏之前先做 Unicode 归一化，每个外部动作都做工具调用校验。这是底线。它挡掉简单攻击，成本几乎为零。

规模超出这套之后，升级顺序是：先上更强的护栏模型（Llama Guard 3，或者调好严重级别的 Azure Content Safety），而不是先加层数；延迟抱怨变严重时再上按块的流式校验；阈值按你自己生产流量的分布来调，而不是按厂商的评测集来调；把 `flagged` 这个布尔值当调试信号，策略决策走 `category_scores`，这样模型更新时不用重新发版就能重标定。

不该部署的，是五个护栏串行堆叠、每个加 100 ms，只因为每家厂商的博客都在卖自己那一个。Constitutional Classifiers 那个结果——95% 的越狱拦截率、0.38% 的误拒率上升——来自一对训练良好的输入与输出分类器，不是来自七层堆栈。[^1] 把力气集中在正确的两层上，胜过在整条管线里撒下一层层薄弱的东西。

下一章 [滥用防治](./03-abuse-prevention.md) 讲基础设施层，怎么让攻击者没法连着打一百万次绕过尝试。

## 参考文献

[^1]: Anthropic Safeguards Research Team, "Constitutional Classifiers: Defending against universal jailbreaks," Anthropic Research, 3 February 2025, https://www.anthropic.com/research/constitutional-classifiers; Sharma et al., arXiv:2501.18837, https://arxiv.org/abs/2501.18837

[^2]: OpenAI, "Moderation," OpenAI API Documentation, accessed June 2026, https://platform.openai.com/docs/guides/moderation

[^3]: OpenAI, "Upgrading the Moderation API with our new multimodal moderation model," 26 September 2024, https://openai.com/index/upgrading-the-moderation-api-with-our-new-multimodal-moderation-model

[^4]: Meta AI, "Llama Guard 3-8B Model Card," GitHub/PurpleLlama, July 2024, https://github.com/meta-llama/PurpleLlama/blob/main/Llama-Guard3/8B/MODEL_CARD.md

[^5]: SB Intuitions, "Compact and Efficient Safeguard for Human-AI Conversations," arXiv:2411.17713, November 2024, https://arxiv.org/html/2411.17713v1

[^6]: Microsoft, "What is Azure AI Content Safety?", Microsoft Learn, last updated January 2026, https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview

[^7]: William Hackett et al., "Bypassing Prompt Injection and Jailbreak Detection in LLM Guardrails," arXiv:2504.11168, April 2025, https://arxiv.org/html/2504.11168v2

[^8]: Aditi Bodhankar, "Stream Smarter and Safer: How NVIDIA NeMo Guardrails Enhance LLM Output Streaming," NVIDIA Developer Blog, 23 May 2025, https://developer.nvidia.com/blog/?p=100628

[^9]: Pride Kavumba et al., "Predict, Don't React: Value-Based Safety Forecasting for LLM Streaming (StreamGuard)," arXiv:2604.03962, April 2026, https://arxiv.org/html/2604.03962

[^10]: Tianpan, "Designing AI Safety Layers That Don't Kill Your Latency," April 2026, https://tianpan.co/blog/2026-04-16-safety-layer-latency-guardrails-design
