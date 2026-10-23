---
title: "零样本与少样本"
description: "示例何时值回它占的 token，何时不值。判断规则、过度提示的悬崖，以及提示词缓存如何改写这笔账。"
slug: 01-zero-shot-and-few-shot
part: 3
chapter: 1
difficulty: beginner
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-2-working-with-model-apis/00-calling-models
  - part-2-working-with-model-apis/07-prompt-caching
  - 00-anatomy-of-a-production-prompt
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 零样本与少样本

一个带五个 200 token 示例的少样本提示词，会给你发出的每一个请求加上 1,000 个 token。在 Claude Sonnet 4.6 上（每百万输入 token $3.00，2026 年 6 月）每天 1 万次请求，这些示例一天要 $30，一年 $10,950。这还是在模型读到一个用户字词之前。多数团队加了示例，从不测它们有没有用，也从没注意到账单上的这一行。

民间经验说示例越多越好。2025 年的实证记录说的正相反：k 一过某个很小的数，准确率就爬到 5 到 25 个示例附近的峰值，然后下降，有时跌得很陡。[^1][^2] 而在现代指令调优模型上，示例往往根本抬不动推理；它们只对齐格式。[^3] 所以问题不是“我该不该加示例”，而是“这几个具体的示例值不值它们那些 token，如果值，我能不能不再每个请求都为它们付一遍钱”。

## 同一个旋钮的两端

零样本与少样本不是两种独立技术，而是同一个刻度盘的两端：你用文字说明了任务的多大一部分，又用示范说明了多大一部分。Tom Brown 的 GPT-3 论文在 2020 年把这些点定了名。零样本只给指令；单样本给一个（输入，输出）示范；少样本给 k 个，k 通常是 2 到 16。[^4]

三种情况下机制都一样。模型不会在请求之间学到任何东西。它对整个提示词跑一次前向传播，示范就放在上下文里紧挨着查询，被同一套注意力模式关注。它们靠“在那儿”来影响输出。这就是人们叫它上下文学习（in-context learning）的原因，也正是你每加一个示例就得在每个请求、每一轮都付 token 的原因——直到你做点什么。

正确的起点是零样本。现代指令调优模型（Claude 4+、GPT-5、Gemini 2.5+、Qwen2.5+）在 RLHF 阶段已经吸收了足够多的任务模板，一条清楚的指令通常就够用。Anthropic 的官方指导说得很直接：从零样本开始，输出不对时再加示例。[^5] OpenAI 的文档说的一样。[^6] 只有在零样本正以可度量的方式白扔钱时，你才升级到少样本。

## 什么时候该升级，示例真正修的是什么

零样本以四种具体方式之一失败时，就加示例。不是“答案感觉有点偏”，那是指令的问题。示例是对的工具，出现在这几种情形：

- **输出格式不对。** 你要特定的 JSON 形状、自定义 XML 标签、某种 markdown 结构，模型老是漂。两三个示例比任何指令都更快把这事定下来。
- **标签词表是自定义的。** 你的分类是 `Triaged / Watching / Escalated`，不是模型见过十亿遍的标准 `Positive / Negative / Neutral`。示例能直接教会这个标签空间。
- **语气或长度系统性跑偏。** 你想要一句话，模型写三段；你想要随意，它写正式。示例校准语域（register）比指令里的形容词可靠得多。
- **有些边界情况用指令说不干净。**“输入含糊时归类为 Neutral，而不是 Unknown”写成散文很别扭，一个示例就看明白了。

注意这份清单上缺了什么：推理。对现代强模型来说，示例并不能可靠地提升推理。Cheng 等人（EMNLP 2025 Findings）在 Qwen2.5 上测过这件事：加入传统的思维链范例，推理并没有比零样本 CoT 更好。他们发现模型“倾向于忽略这些范例，主要盯着指令”。[^3] 如果你的问题是“模型想得不够狠”，少样本就是拧错了旋钮。那属于[思维链、ReAct、自我一致性](02-chain-of-thought-react-self-consistency.md)的地盘。

## 过度提示的悬崖

这里有一件让很多人意外的事。示例多并不严格更好。过了那个很小的峰值，准确率就往下走，有时掉得厉害。

2025 年两项独立研究把这件事钉死了。Oskooei 等人跑了 9 万多条代码翻译，示例数从 0 到 625（上下文最多 800k token）。“功能正确率始终在少样本提示（5 到 25 个示例）时达到峰值”，再往上就明显退化。他们把这个效应叫作“多示例悖论”（many-shot paradox）。[^1] Tang 等人在一个完全不同的任务上确认了同一点——跨 GPT-4o、GPT-3.5-turbo、LLaMA-3.1-8B、Gemma-3-4B 的领域特定文本分类，峰值落在 5 到 20 个示例之间，之后明显下降。[^2] 模型越小，峰值越低。在他们的测试里，LLaMA-3.2-3B 超过 100 个示例后塌成只输出一个标签。

![一条山形准确率曲线：左侧从零样本基线上升，中间在 5 到 25 个示例处到顶，右侧落入标注出的过度提示区；峰值上方有一小段绿色区间，标出 Anthropic 建议的 3 到 5 个](../../../ai-eng/part-3-prompt-engineering/images/01-over-prompting-curve.png)
*准确率随示例数量的变化，来自两项独立的 2025 年研究。峰值在 5 到 25 之间；过了这个点，更多示例只会更糟。*

所以实用的默认值很小。Anthropic 的指导对多数任务落在 3 到 5 个示例。[^5] 在你动手测量之前，上限大约 20。过了这个数，你不是在改进模型，而是在为有害的 token 付钱。如果你确实需要很多示例（比如长尾标签空间），那就在 k = 0、3、5、10、20 上测准准确率再定。峰值是真的，而且比多数工程师猜得低。

## 挑示例：随机、检索还是人工精选

一旦定下小 k，你就得决定选哪 k 个。按投入从低到高：

- **人工精选且固定。** 一组手挑的 3 到 5 个示例，跟提示词一起发出去。运维最省、最好缓存，对多数格式和语气任务足够用。这是默认做法。
- **TF-IDF 检索。** 每个请求用经典词法相似度从池子里取最相似的 top-k。确定性强、便宜。Tang 等人发现，在领域特定任务上，TF-IDF 检索与语义检索旗鼓相当。[^2]
- **语义检索（KATE 风格）。** 把每个候选示例和查询都做 embedding，按余弦相似度取 top-k。Liu 等人（2021）表明，在 GPT-3 上基于检索的选择相比随机选，在表格转文本生成上相对提升 41.9%，在开放域问答上相对提升 45.5%。[^7] 输入差异大、而且你手里有标注池时，值得上。

检索版实现很直白：

```python
from dataclasses import dataclass

@dataclass
class Example:
    input_text: str
    output_text: str
    embedding: list[float]  # 预先算好

def cosine(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    na = sum(x * x for x in a) ** 0.5
    nb = sum(x * x for x in b) ** 0.5
    return dot / (na * nb) if (na and nb) else 0.0

def select_examples(query_emb: list[float], pool: list[Example], k: int = 5) -> list[Example]:
    # 升序排序，让最相似的示例落在列表最后。
    return sorted(pool, key=lambda e: cosine(query_emb, e.embedding))[-k:]
```

池子本身有两条不那么显然的规则。第一，多样性和相似性一样重要。五个示例都在示范同一个边界情况，那教会的就是那一个情况，别的什么也没教。kNN 检索之后，检查一遍所有标签类别是否都被覆盖、有没有发出去一堆近似重复的样本。Anthropic 的文档明确警告：“覆盖边界情况，并且变化足够多，免得 Claude 学到意料之外的模式。”[^5] 第二，示例必须来自与生产输入相同的分布。你用干净正式的英文写的示例，会在生产输入以带错别字和 emoji 的口语化用户语气到来时，把模型带偏。

## 顺序的影响比你想的更大

Zhao 等人（ICML 2021）揭示了上下文学习中一件让人不舒服的事：同一个模型、同一批示例、同一个查询，仅凭示例顺序不同，准确率就能从接近随机摆动到接近当前最好水平。他们的校准修正方案在分类任务上最多找回了 30 个百分点的绝对准确率。[^8] 模型对上下文里看到的东西有偏好，而这些偏好会左右答案。

两种偏好造成了这种摆动。**近因偏好（recency bias）**：模型偏向最后一个示例的标签。**多数标签偏好（majority-label bias）**：模型偏向出现次数最多的那个标签。于是有三条经验法则：

1. **平衡各标签的数量。** 如果你要展示覆盖三个类别的五个分类示例，别让其中四个都是“Positive”。模型会继承这个先验。
2. **把标签打散。** 别把所有“Positive”堆在一起再跟一堆“Negative”。交替着放。
3. **把最相关的示例放最后。** 如果你用检索，相似度最高的那个放最底下，离查询最近。这样近因偏好就站在你这边，而不是跟你作对。

唯一可以打破第三条规则的地方，是打乱顺序会毁掉前缀缓存的时候。这就说到了账单。

## 示例要花多少钱，缓存又如何改写这笔账

没有缓存时，每个示例在每个请求上都要付一次钱。五个 200 token 的示例 = 提示词里多 1,000 token。在 Sonnet 4.6 上（每百万输入 $3.00，2026 年 6 月）每天 1 万次请求，光示例就是一天 $30、一年 $10,950。示例数翻倍，账单就翻倍。这笔账是残忍的线性：`daily_cost = (example_tokens / 1_000_000) * input_price * requests_per_day`。

前缀缓存打破这个线性。服务商在请求之间留着你那段稳定前缀的 KV 张量；缓存 TTL 内的后续请求为这段前缀付大约 10% 的输入价，而不是全价。在 Anthropic 的 5 分钟档，写入是基础价的 1.25 倍（Sonnet 4.6 上每百万 $3.75），读取是 0.10 倍（每百万 $0.30）；OpenAI 上是自动的，读取价 10%；Gemini 上你得显式创建一个缓存资源。[^9][^10][^11] 完整机制见[提示词缓存](../part-2-working-with-model-apis/07-prompt-caching.md)。

我们这 1,000 token 的示例块，每天 1 万次请求，缓存每 5 分钟预热一次（一天 288 个窗口）：

```python
EXAMPLE_TOKENS = 1_000
REQUESTS = 10_000
WINDOWS = 288  # 一天里的 5 分钟窗口数

BASE  = 3.00 / 1_000_000   # Anthropic Sonnet 4.6 输入价，2026 年 6 月
WRITE = 3.75 / 1_000_000   # 5 分钟缓存写入
READ  = 0.30 / 1_000_000   # 缓存读取

no_cache  = EXAMPLE_TOKENS * BASE * REQUESTS
with_cache = (EXAMPLE_TOKENS * WRITE * WINDOWS
              + EXAMPLE_TOKENS * READ * (REQUESTS - WINDOWS))

print(f"No cache:   ${no_cache:.2f}")    # $30.00
print(f"With cache: ${with_cache:.2f}")  # $3.99
```

示例账单砍掉 87%，代价只是加一个配置字段。回本几乎是立刻的：在 TTL 窗口内大约两次请求之后，缓存就把写入的溢价赚回来了。在任何生产流量水平上，只要示例块稳定，缓存都是净赚。

问题在于“稳定”是什么意思。缓存以这段前缀字节的哈希为键。改一个示例、改一下顺序、在指令里换一个词，哈希就变了，你得重写一份缓存。所以这一章里天然有一处张力：

- **固定的精选示例**缓存得非常漂亮。每次请求前缀都一样，87% 的节省全额拿到。
- **按请求检索的示例**会毁掉缓存。每个查询的 top-k 不同，每个前缀都唯一，每个请求都付全价。Lumer 等人（2026）在生产里发现了这一点：对动态内容做天真的全上下文缓存，反而可能*增加*延迟，因为你付了缓存写入的溢价，却一次也没命中缓存。[^12]

既想要检索质量又想要缓存节省时，出路是混合前缀。把一批固定的精选示例放在前面缓存起来。检索出的示例注入到缓存断点之后、用户消息里——那部分按全价输入计费，但大得多的系统提示词能拿到缓存的读取价。你拿不到完整的 87%，但也不至于全丢。Lumer 等人明确推荐这个形状：“把动态内容放在系统提示词末尾……比天真的全上下文缓存带来更稳定的收益。”[^12]

## 默认做法

从零样本开始。测。如果输出格式不对、标签是自定义的、语气飘，或者有散文说不清的边界情况，就往系统提示词里加 3 到 5 个手挑的示例。平衡标签、打散排列、把最匹配的放最后。测量之前最多到 20。如果你要上检索，就留一个静态库缓存着，把检索出的示例注入到缓存断点之后。

默认加示例、而且从不删，是本能。它们感觉免费。它们不免费。每个示例都是账单上的一行，每个请求都自己打印一遍；而过了一个很小的 k，这一行还在扣你的准确率。纪律是把示例纳入预算，就像你对任何其他生产资源做的那样，并且在下个模型版本让零样本悄悄变强时，重新审视这份预算。

## 参考文献

[^1]: Amirkia Rafiei Oskooei et al., "When Many-Shot Prompting Fails: An Empirical Study of LLM Code Translation", ICSE 2026 RECODE workshop, arXiv:2510.16809, https://arxiv.org/abs/2510.16809
[^2]: Yongjian Tang et al., "The Few-shot Dilemma: Over-prompting Large Language Models", arXiv:2509.13196, September 2025, https://arxiv.org/abs/2509.13196
[^3]: Xiang Cheng et al., "Revisiting Chain-of-Thought Prompting: Zero-shot Can Be Stronger than Few-shot", EMNLP 2025 Findings, arXiv:2506.14641, https://arxiv.org/abs/2506.14641
[^4]: Tom B. Brown et al., "Language Models are Few-Shot Learners", NeurIPS 2020, https://arxiv.org/abs/2005.14165
[^5]: Anthropic, "Use examples (multishot prompting) to guide Claude's behavior", Anthropic API Docs, https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting (fetched June 12 2026)
[^6]: OpenAI, "Prompt engineering", OpenAI Platform Docs, https://platform.openai.com/docs/guides/prompt-engineering (fetched June 12 2026)
[^7]: Jiachang Liu et al., "What Makes Good In-Context Examples for GPT-3?", DeeLIO 2022 workshop, arXiv:2101.06804, https://arxiv.org/abs/2101.06804
[^8]: Zihao Zhao, Eric Wallace, Shi Feng, Dan Klein, Sameer Singh, "Calibrate Before Use: Improving Few-Shot Performance of Language Models", ICML 2021, https://proceedings.mlr.press/v139/zhao21c.html
[^9]: Anthropic, "Prompt caching", Anthropic API Docs, https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching (fetched June 12 2026)
[^10]: OpenAI, "Prompt caching", OpenAI Platform Docs, https://platform.openai.com/docs/guides/prompt-caching (fetched June 12 2026)
[^11]: Google, "Context caching", Gemini API Docs, https://ai.google.dev/gemini-api/docs/caching (fetched June 12 2026)
[^12]: Elias Lumer et al., "Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks", arXiv:2601.06007, January 2026, https://arxiv.org/abs/2601.06007
