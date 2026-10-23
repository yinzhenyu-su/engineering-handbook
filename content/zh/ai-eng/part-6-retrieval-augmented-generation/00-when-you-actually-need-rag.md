---
title: "到底什么时候才真的需要 RAG"
description: "把 RAG、长上下文、微调当成一道成本题：长上下文按查询付费，RAG 建一次索引反复用。默认规则、例外情况，以及 RAG 反而是错答案的时候。"
slug: 00-when-you-actually-need-rag
part: 6
chapter: 0
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-1-how-models-behave/03-embeddings
  - part-2-working-with-model-apis/07-prompt-caching
  - part-5-context-engineering/00-context-is-more-than-the-prompt
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# 到底什么时候才真的需要 RAG

你手上有 60 万词元的语料和一个问题。按 2026 年 6 月的 GPT-5.4 价格，把整份语料塞进上下文窗口，一次查询约 1.51 美元；取回最相关的五个片段（约 3000 词元）再据此作答，约 0.015 美元。同一个问题，同一个模型。账单差两个数量级。[^1][^2][^3]

这个差距就是本章存在的全部理由。前沿模型如今都带百万词元的上下文窗口，每有一个新的长上下文模型落地，就有人宣布 RAG 已死。窗口是真的；成本结构没变。长上下文按查询付费，费用与语料大小成正比。检索增强生成（retrieval-augmented generation，RAG，即查询时取回相关片段塞进提示词的这套做法）只在建索引时付一次钱，之后只为取回的那部分付费。微调（fine-tuning）付一大笔一次性训练成本，把知识烧进权重里。三者之间怎么选，主要是经济问题，由三个数字决定：语料多大、多久变一次、会有多少次查询打到它上面。

## 三种成本形态

拿一份 60 万词元的知识库、GPT-5.4 的价格、500 词元的提问和 400 词元的回答算一遍：

```python
# 单次查询成本：长上下文 vs RAG（GPT-5.4，2026 年 6 月价格）
GPT54_INPUT  = 2.50   # 美元/百万词元
GPT54_OUTPUT = 15.00  # 美元/百万词元
EMBED_SMALL  = 0.02   # 美元/百万词元，text-embedding-3-small

def long_context(corpus_tokens, prompt_tokens, output_tokens):
    inp = (corpus_tokens + prompt_tokens) / 1_000_000 * GPT54_INPUT
    out = output_tokens / 1_000_000 * GPT54_OUTPUT
    return inp + out

def rag(retrieved_tokens, prompt_tokens, output_tokens, query_tokens):
    embed = query_tokens / 1_000_000 * EMBED_SMALL
    inp = (retrieved_tokens + prompt_tokens) / 1_000_000 * GPT54_INPUT
    out = output_tokens / 1_000_000 * GPT54_OUTPUT
    return embed + inp + out

lc  = long_context(600_000, 500, 400)        # 约 $1.51
r   = rag(3_000, 500, 400, 50)               # 约 $0.015
print(f"LC ${lc:.4f}  RAG ${r:.4f}  ratio {lc/r:.0f}x")
```

LightOn 在 2025 年 11 月做过一次类似的对比（1000 页、约 60 万词元的知识库，每天 100 次请求），结论是典型负载下 RAG 比长上下文便宜 8 到 82 倍，延迟也更好。[^1] 这个倍数区间在提示词缓存（prompt caching）面前会收窄：Claude Sonnet 4.6 上缓存读取是每百万词元 0.30 美元，标准价 3.00 美元，输入部分打一折，但只有各次请求的语料完全一致时才成立。[^2] 语料静态、同一条提示词反复用，缓存能填掉大部分差距。语料只要会动，一动就得按全价付输入。

RAG 这边的索引成本几乎是个零头。用 `text-embedding-3-small` 按每百万词元 0.02 美元给 60 万词元的语料做嵌入，只花 0.012 美元；托管向量数据库每月起价约 45 到 50 美元（Weaviate Cloud 的 Flex 套餐、Pinecone Standard 的月度最低消费），账单随索引大小和查询量增长。[^3][^4] 只要查询量稍微像样，第一天就回本。

![一张按查询成本横向对比三种策略的图。长上下文那条柱子高得夸张，标着 $1.51。RAG 那条柱子在底部薄成一条线，标着 $0.015。微调那条柱子很短，上方叠了一个方块，表示一次性的训练成本。](../../../ai-eng/part-6-retrieval-augmented-generation/images/00-cost-shapes.png)
*三种成本形态。长上下文每次查询都为整份语料付账；RAG 只为取回的片段付费，加上接近零的索引摊销；微调先付一大笔训练钱，之后推理永久便宜。*

## 默认规则

**语料超过 10 万词元，或者更新频率快于每月一次，就用 RAG。** 规则就这一条。低于阈值、语料静态时，长上下文配提示词缓存，在准确率和开发负担上都赢。任一阈值之上，检索在成本结构和运维理智上赢。

10 万这个数是实用值，不是什么魔法数字。低于它，单个前沿模型的上下文能宽裕地吞下整份语料，还可以靠提示词缓存把输入成本摊到多次查询上。高于它，你要么每次调用都付全价输入（一个月几百次查询以上就撑不住），要么就得接受注意力摊薄在几十万词元上带来的准确率下降。

30 天更新阈值说的是反方向的索引经济性。语料每天都变，微调直接出局，因为重新训练的周期跟不上。长上下文在新鲜度上没问题，但在体量上很贵。RAG 处理增量更新很干净：只重嵌改过的文档，写进索引，下一次请求就能看到新内容。

剩下三个维度决定具体怎么选：

- **语料大小。** 5 万词元以下，放进系统提示词配缓存完全够。5 万到 20 万是灰色地带，由缓存效率和查询量定。20 万以上，除非有强理由，默认 RAG。
- **更新频率。** 多年不变，微调可以上桌。按月或更慢，长上下文加缓存依然便宜。按周或更快，RAG 变成必须。
- **查询类型。** 查找型查询（“我们的带薪休假政策是什么？”）是 RAG 的形状。整体型查询（“总结这份 100 页合同的主题”）是长上下文的形状，因为它们没法拆成片段。

同样的逻辑写成能跑的代码：

```python
from dataclasses import dataclass
from enum import Enum

class Strategy(Enum):
    RAG = "rag"
    LONG_CONTEXT = "long_context"
    FINE_TUNE = "fine_tune"

@dataclass
class CorpusProfile:
    size_tokens: int
    update_frequency_days: int
    queries_per_month: int
    fraction_holistic: float  # 需要整篇综合作答的查询占比

def decide(p: CorpusProfile) -> Strategy:
    fits_in_context = p.size_tokens < 200_000
    updates_often = p.update_frequency_days < 30
    holistic_heavy = p.fraction_holistic > 0.5

    if updates_often and p.size_tokens > 200_000:
        return Strategy.RAG
    if fits_in_context and holistic_heavy and p.queries_per_month < 500:
        return Strategy.LONG_CONTEXT
    if not updates_often and p.size_tokens < 50_000:
        return Strategy.FINE_TUNE
    return Strategy.RAG

print(decide(CorpusProfile(1_000_000, 7, 5_000, 0.2)))   # RAG
print(decide(CorpusProfile(80_000, 180, 100, 0.8)))       # LONG_CONTEXT
print(decide(CorpusProfile(20_000, 365, 10_000, 0.1)))    # FINE_TUNE
```

## 准确率数据到底说了什么

“RAG 已死”派通常只举一篇论文：Google DeepMind 在 EMNLP 2024 的研究显示，只要“资源给足”，长上下文在 LongBench 和 InfiniteBench 上就比 RAG 高 7.6%（Gemini-1.5-Pro）和 13.1%（GPT-4o）。[^5] 这个数字是真的。但它只是故事的一半。

同一篇论文提出了一个叫 Self-Route 的混合做法：先问 RAG，允许它拒答，只有模型说凭取回的片段答不了时才升级到长上下文。对 Gemini-1.5-Pro，81.74% 的查询能靠 RAG 答出来；系统在只用了 38.4% 词元的前提下，准确率与长上下文相差不到 2.2 个百分点。[^5] 标题数字说的是长上下文赢；工程上的结论是，你可以用三分之一的成本把这场胜利几乎全买回来。

LaRA 基准（ICML 2025，2326 个测试用例，覆盖小说、财务报表和学术论文）把图景描得更清。32K 上下文时，长上下文平均比 RAG 高 2.4%。到 128K，开源模型上趋势反转：RAG 领先 3.68%。[^6] 长上下文处理能力更强的闭源模型（GPT-4o、Claude 3.5 Sonnet、Gemini 1.5 Pro）在两种长度下依然占优，但差距既取决于模型强度，也取决于窗口大小。有两类任务分化得很明显：

- **对比与综合类任务**：长上下文领先 14 到 15 个百分点。比较两份合同、总结主题、“v1 到 v2 之间改了什么”，都不是检索形状的问题。它们需要一次看到整份文档。
- **幻觉检测**：所有规模的模型上 RAG 都决定性胜出。当问题是“这个说法在原文里出现过吗”，检索出相关段落、让模型基于它作答，胜过指望模型自己在 128K 词元里找到那根针。[^6]

控制什么东西进入上下文还有第二个理由，而且比成本论证更早。Liu 等人在 2024 年指出，当相关文档位于 20 篇文档提示词的中间、而不是开头或结尾时，GPT-3.5-Turbo 的多文档问答准确率会掉 20 多个百分点。最差的情况下，中间位置的成绩（20 篇里排第 10，52.9%）低于闭卷成绩（56.1%），也就是说模型在完全不给文档时反而更好，把正确文档埋在中间只会更差。[^7] GPT-4 呈同样的 U 形曲线，只是绝对准确率更高。RAG 绕开了这个问题：什么进窗口、放在哪儿由你决定——排在前面的片段放开头（首因）或结尾（近因），绝不埋在中间。

## RAG 反而是错答案的时候

默认规则有四类真实例外，其中之一成立时还硬上 RAG，就是团队把几个月砸在自己根本不需要的基础设施上的典型方式。

**语料小且稳定。** 一份 3 万词元、一年没动过的风格指南不需要向量数据库。丢进系统提示词、打开提示词缓存，收工。Anthropic 在 Sonnet 4.6 上每百万缓存读取 0.30 美元，前缀一旦热起来，单次查询成本可以忽略。[^2] 你省掉了分块（chunking）、嵌入流水线、检索评测，还有一个随时待命的向量库。

**对单份文档做整体推理。** “总结这份 100 页合同的主题”或者“这两份规范之间改了什么”没法拆成分片检索。整份文档就是输入。Liu 等人发现阅读器在取回 20 篇文档附近性能就饱和了，再多给只带来约 1.5% 的提升，却要付出巨大的上下文成本。[^7] 任务需要整份文档，就别检索，直接装进去。

**问题在行为，不在知识。** 如果模型输出格式不对、缺少某种领域特有的语气，或者在训练数据里明明充分的场景下失手，RAG 只会加上检索噪声，治不了根因。微调才是对的工具。判断信号是：你的评测失败集中在风格、结构或任务遵循上，而不是缺事实。（OpenAI 已于 2026 年 5 月对新用户关停托管微调平台，所以 GPT 系列模型得转向 Anthropic、Google 或开源方案。[^8]）

**静态语料上查询量极低。** 一个月不到十次查询打在一份冻结的语料上，为了它跑向量数据库、嵌入流水线和监控的运维开销，超过你在单次查询成本上省下的那点钱。直接把语料发过去。

一个有用的反例，用来打“RAG 永远是错的”这个立场：Claude Code 基于 grep 的代码导航之所以能用，是因为代码在词法上可查。函数名、类型签名、导入路径都是精确字符串。企业知识库（PDF、扫描报告、多语言内容、图表）不是。用 LightOn 的话说：你没法 grep 一张图。[^1] 查询和文档共享词汇时词法搜索赢；不共享时基于嵌入的检索赢。

## 决策规则抓不住的两类失效

有两个模式是在战略定完之后才在生产里冒出来的，在你因为一个边缘案例就拍板 RAG 之前，值得先标出来。

第一个是多跳推理（multi-hop reasoning）。标准 RAG（嵌入查询、取回 top-k 片段、生成）在“要拿第一步的答案去构造第二步的查询”时会崩。检索器只看得到原始问题；如果歌手的名字不在问题里，它就取不到“X 这首歌的演唱者的国籍”。这是多跳问答基准上最常见的 RAG 失效，也是 Self-Route 转投长上下文的单一最大类别。[^5] 如果你的查询要在多份文档之间链式推理，单纯的先检索后生成不够；你需要迭代检索，或者一条混合升级路径。

第二个是把“语料每天更新”当成纯粹索引侧的问题。重嵌本身花不了多少钱。运营一个每天有 10% 语料在变的向量数据库，不是这么回事。索引写放大、删除再插入的审计日志、入库延迟，都在裸算力之上加成本，而这些在只按查询量做的容量估算里一个都不会出现。按入库峰值做容量规划，而不是只按查询峰值，这就是第 90 天还能正常跑的 RAG 系统和凌晨三点要紧急重建索引的系统之间的区别。

这两个问题都是战略定了之后，在[分块与索引](./01-document-processing-and-chunking.md)和[检索策略](./02-embeddings-and-vector-search.md)里解决的。它们不是语料和更新形态说你需要 RAG 时该躲开 RAG 的理由；它们是你要老老实实做容量规划和设计的理由。

放到架构尺度上，HLD 第 9 部分的 [AI 系统设计](../../hld/part-9-ai-ml-system-design/01-rag-pipelines.md)覆盖了企业 RAG 这个案例：入库流水线、向量库容量估算、可观测性，也就是本章刚做完的那道战略决策之下所有东西的白板视图。

## 参考文献

[^1]: LightOn，《RAG is Dead, Long Live RAG: Retrieval in the Age of Agents》，基于 Amélie Chatelain 在 Weights & Biases Fully Connected 2025 的演讲，2025 年 11 月 12 日（存档副本；原页面已下线），https://web.archive.org/web/20260410222220/https://lighton.ai/fr-blog-posts/rag-is-dead-long-live-rag-retrieval-in-the-age-of-agents；成本计算器：https://github.com/AmelieChatelain/long-context-surely-killed-RAG
[^2]: Anthropic，《Claude Pricing - API》，2026 年 6 月核实，https://claude.com/pricing
[^3]: OpenAI，《API Pricing》，2026 年 6 月核实，https://openai.com/api/pricing/ 以及《text-embedding-3-small model page》，https://developers.openai.com/api/docs/models/text-embedding-3-small
[^4]: Weaviate，《Pricing》，https://weaviate.io/pricing；Pinecone，《Understanding cost》，https://docs.pinecone.io/guides/manage-cost/understanding-cost
[^5]: Zhuowan Li 等，《Retrieval Augmented Generation or Long-Context LLMs? A Comprehensive Study and Hybrid Approach》，EMNLP 2024 Industry Track（Google DeepMind），https://arxiv.org/abs/2407.16833
[^6]: Kuan Li 等，《LaRA: Benchmarking Retrieval-Augmented Generation and Long-Context LLMs》，ICML 2025（Alibaba-NLP），https://arxiv.org/abs/2502.09977
[^7]: Nelson F. Liu 等，《Lost in the Middle: How Language Models Use Long Contexts》，TACL 12:157-173，2024，https://arxiv.org/abs/2307.03172
[^8]: OpenAI，《Introducing vision to the fine-tuning API》（2026 年 5 月 8 日的说明，宣布关停微调平台），https://openai.com/index/introducing-vision-to-the-fine-tuning-api/
