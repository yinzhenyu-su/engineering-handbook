---
title: "混合检索与重排序"
description: "为什么在精确词项上 BM25 仍然胜过向量检索、如何用 RRF 把两者融合，以及重排序模型什么时候才配得上它带来的延迟。"
slug: 03-hybrid-search-and-reranking
part: 6
chapter: 3
difficulty: intermediate
prerequisites:
  - 02-embeddings-and-vector-search
  - part-1-how-models-behave/03-embeddings
tags: [llm, rag, search]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 混合检索与重排序

用户在客服搜索框里敲下 `E7-492-B`。这是一个固件错误码，上周二你索引的三篇文档里刚开始出现它。稠密检索器（dense retriever）返回了五篇讲恒温器电池的文章，没有一篇提到这个码。而那个精确的字符串就躺在你的索引里，原封未动，排在第 47 位。

这就是把纯向量 RAG 在生产里搞死的失败模式。双塔编码器（bi-encoder）把每个文本块压成一个 1,024 维向量。这种压缩在“skincare routine”匹配“moisturizer guide”这类事情上很好用，但要把一个在编码器训练数据里出现过 0 次的 token 原样保住，它很糟糕。编码器把 `E7-492-B` 嵌到某个读音相近、却与文档毫无语义关系的 token 附近，剩下的交给余弦相似度去办。[^1]

解法不是换一个更好的嵌入模型，而是再加一个压根不做压缩的检索器。

## 为什么在精确词项上 BM25 依然赢

BM25 是 1994 年出自伦敦城市大学（City University London）的一个关键词排序函数。32 年过去，年年都有人说“它随时会被取代”，而它至今仍跑在你用过的几乎每一个生产搜索系统里。[^2] 原因是一个稠密向量复制不了的性质：零表示漂移。语料库里每一个不同的 token，在倒排索引（inverted index）里都有自己的条目。新上线的 SKU、新的错误码、新的人名：周二建好索引，周三就能精确命中。

某个词项在某篇文档里的得分，短到可以写在餐巾纸上：

```python
import math

def bm25_score(tf, df, n_docs, dl, avgdl, k1=1.2, b=0.75):
    """One term, one document. Sum across query terms for the full score."""
    idf = math.log((n_docs - df + 0.5) / (df + 0.5) + 1)
    tf_norm = (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * dl / avgdl))
    return idf * tf_norm
```

整个设计就靠两个参数。`k1=1.2` 让词频饱和：一个词在文档里第 10 次出现，得分远低于第 1 次，所以那种把查询重复 50 遍的垃圾页面压不住场子。`b=0.75` 做长度归一：一份 50 页的文档不会只因为足够长、碰巧包含了稀有词项，就白捡一分。[^3] 这两个默认值已经撑了二十年，你几乎从来不需要调。

BM25 做不到的是同义。搜“car”，漏掉“automobile”；搜“skincare routine”，漏掉那篇题为“moisturizer guide for sensitive skin”的文章。2026 年 5 月一份针对 1,000 个商品目录的 Elasticsearch 案例研究，把这个差距直接量了出来：在混合意图的查询上，单用 BM25 的 Recall@10 = 0.43；加上稠密检索、把两路结果融合，这个数字被推到 0.75。[^4] 相关文档本来就在索引里，只是 BM25 隔着词汇不匹配（vocabulary mismatch）看不见它们。

所以你两路都跑。BM25 抓住精确 token，稠密检索器抓住改写。接下来你多出一个新问题：两个排好序的列表怎么合。

## 倒数排名融合（RRF）：最便宜又管用的做法

最朴素的做法是把分数相加。这会坏掉。BM25 产出的是没有上界的正数，量纲取决于语料库；余弦相似度则被限在 `[-1, 1]` 里。一个典型的 BM25 命中可能得 12.4 分，一个典型的余弦命中得 0.84。取个平均，BM25 默认就赢，哪怕它不该赢。[^5]

你可以做归一化、加权、重新校准。也可以干脆把分数全扔掉，只用排名。这就是倒数排名融合（Reciprocal Rank Fusion，RRF），Cormack、Clarke 和 Buttcher 在 SIGIR 2009 上提出，17 年过去，它仍是 Elasticsearch、OpenSearch、Qdrant、Pinecone 和 Azure AI Search 的默认做法。[^6][^7]

公式只有一行：

```python
from typing import Dict, List

def rrf_fuse(ranked_lists: List[List[str]], k: int = 60) -> List[tuple]:
    """Fuse N ranked lists by rank position alone. Cormack et al., SIGIR 2009."""
    scores: Dict[str, float] = {}
    for ranked in ranked_lists:
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

一篇文档在两个列表里都排第一，得分 `2/61 = 0.0328`；只在一个列表里排第一、另一个列表里没有，得分 `1/61 = 0.0164`。常数 `k=60` 直接来自原论文，此后在所有实现里都成了默认值；只有当别的增益都被你榨干之后，才轮到调它。[^7]

![一张左右对照图：两个排序好的检索列表通过倒数排名融合合并，左侧是 BM25 结果、右侧是稠密向量结果，两者一起汇入中间按 RRF 分数排序的融合列表；有一篇文档同时出现在两个列表里，最终升到合并列表的顶部，说明在两个检索模式里都排名靠前的文档会被放大](../../../ai-eng/part-6-retrieval-augmented-generation/images/03-rrf-fusion.png)
*两个排序列表只按名次合并。在两个检索器里都排得靠前的文档胜出。*

Elasticsearch 团队 2025 年 5 月发布了一个 `linear` 检索器，走的是相反的路子：先用 min-max 把分数归一化，再加权。[^5] 在分数幅度本身携带真实信号的工作负载上，它能胜过 RRF：一个原样出现且频繁出现的查询词项，会产出一个离群的 BM25 高分，而 RRF 把它压平成“排第 1，得分 1/61”。代价是校准。每次批量导入文档，分数分布都会变；一家法律取证平台报告说，导入 20 万份新文件后召回率掉了 30%，因为 BM25 分数猛涨而余弦分数保持稳定，原来的融合权重就这么悄悄失真了。[^8] 如果你既没有留出的评测集，也没有一个每晚重跑网格搜索的任务，RRF 是更安全的默认值——它在构造上就对分数漂移稳健。

## 混合检索什么时候帮倒忙

“永远上混合检索”是共识建议，而它至少在一类重要的语料库上是错的。

Superlinked 的 NirantK 在六份银行 10-K 申报文件上跑了 8 组条件的消融实验（ablation）：1,854 条查询、2,942 页、8,766 条相关性判定，所有数字都公开。[^9] 在这个语料库上，纯向量的 NDCG@10 是 0.396，混合 RRF（BM25 + 向量）是 0.358。混合反而*更差*。金融 10-K 文本密集、技术性强、语义单一，BM25 赖以生存的稀有词项信号几乎不存在，于是把 BM25 加进融合只是加了噪声。RRF 对所有列表一视同仁，一个低质量的排序会稀释高质量的排序——因为差列表里每一篇文档照样贡献非零分数。

结论不是“别做混合”，而是“在你的数据上量一量”。如果你的语料库里有商品编码、人名、错误字符串、版本号，或者任何编码器从没训练过的词汇，混合就会有效，通常能带来 15 到 35 个点的召回提升。[^8] 如果你的语料库是清一色的稠密散文、精确 token 查询很罕见，那就上线前先跑一遍消融；第二个检索器可能纯粹是一笔没有回报的开销。

## 重排序模型与收益递减曲线

混合检索给你一个候选列表，其中大部分是噪声。标准的生产做法是留 25 到 50 个候选，再跑第三阶段，把每个候选与查询直接打分。

交叉编码器（cross-encoder）重排序模型，和检索阶段用的双塔编码器是两种形状。双塔编码器分别嵌入查询和文档、再比较两个向量，所以它快到能在几百万篇文档上跑。交叉编码器把 `[CLS] query [SEP] document [SEP]` 拼起来，对每一对跑一次完整的 transformer 前向。注意力头同时看到两边，于是能识别出词项重叠、否定、以及余弦相似度会抹平的细粒度不匹配。代价是每个查询 O(k) 次前向，所以它只跑在一小份候选列表上，永远不跑整个语料库。[^10]

质量提升大到足以压过 Superlinked 那次消融里其它所有架构选择。纯向量是 0.396 NDCG@10，混合 RRF 是 0.358；在混合候选池上加一个交叉编码器重排序模型（mxbai-rerank-large），直接跳到 0.600，比任何检索侧的微调都涨得多。[^9] 在 MTEB 和 BEIR 基准上，重排序模型典型的增益是 +5 到 +15 个 NDCG@10 点，往往就是“能用的 RAG 系统”和“能答对的 RAG 系统”之间的差别。[^11]

反转在这里：候选给得越多并不越好。

Jacob、Lindgren、Zaharia 等人 2024 年 11 月的一篇论文，把候选集从 10 一路加到 1,000 来跑交叉编码器，得到一条明显的非单调曲线：质量从 10 个候选涨到 50 到 100 个左右，之后越加反而*越差*。[^10] 越过这个阈值，重排序模型“经常给那些与查询既无词汇重叠、也无语义重叠的文档打高分”。模型是在“看着合理但其实是错的”那种负例上训练的；完全不相关的文档属于分布外，模型的置信度于是变成没校准的噪声。

![一张折线图：纵轴是用 NDCG@10 衡量的重排序质量，横轴是传给交叉编码器的候选数量，范围从 10 到 500；曲线从 10 个候选起陡升，在 50 个左右达到峰值，到 100 个时走平，超过 200 后明显下滑，一条水平虚线在峰值下方标出“不用重排序”的基线](../../../ai-eng/part-6-retrieval-augmented-generation/images/03-reranker-curve.png)
*重排序质量在 50 到 100 个候选处见顶，再往上就开始下降。传 500 个候选只会让系统更差，不是更好。*

由此得出的实用规则是：从混合检索里取 25 到 50 个候选交给重排序模型，再把前 5 到 10 个交给 LLM。只有当召回比延迟更重要时（法律取证、合规检索）才推到 100。永远不要把整个语料库丢进去，那是一次 O(N) 前向，把第一阶段之所以要快这件事的意义全毁了。[^12]

```python
# pip install cohere；环境变量里设置 COHERE_API_KEY
from typing import List
import cohere

def rerank_top_k(query: str, candidates: List[str], top_n: int = 5) -> List[dict]:
    """Pass 25-50 candidates from hybrid retrieval. Diminishing returns past ~100."""
    co = cohere.Client()
    response = co.rerank(
        query=query,
        documents=candidates,
        model="rerank-v3.5",
        top_n=top_n,
    )
    return [
        {"index": r.index, "score": r.relevance_score, "text": candidates[r.index]}
        for r in response.results
    ]
```

## 怎么挑重排序模型

重排序模型市场每季度都在变，所以任何具体推荐都有保质期。真正稳定的不是排行榜，而是这道选择题的形状。

| 模型 | 托管方式 | 价格（2026 年中） | 上下文 | 适用场景 |
|---|---|---|---|---|
| Cohere rerank-v3.5 | 托管 API | 每 1K 次查询 $2.00（每次最多 100 篇文档） | 4,096 token | 对延迟敏感的英语优先应用；在 2025 年 11 月的 ELO 基准中最快[^13] |
| Voyage rerank-2.5 | 托管 API | 每 1M token $0.05（约 $0.0025/次请求） | 32,768 token | 对质量敏感的工作负载；长文档；在同一基准中“综合最均衡”[^14] |
| BGE-reranker-v2-m3 | 自托管 | 开放权重 | 8,192 token | 多语言、本地部署、没有按次成本；支持 100 多种语言[^15] |

Voyage 更长的上下文窗口比看上去更重要。4K token 的上限意味着，超长的文档你要么截断（可能正好丢掉相关的那一段），要么切块、分块重排序（多一层复杂度）。面对长 PDF、转录稿或代码文件，32K 窗口这个杠杆比基准上那一两个点要大得多。

一个 110M 参数的交叉编码器，在单张 T4 GPU 上处理 20 个候选大约要 45 到 65 ms，换到 CPU 上会涨到 100 到 300 ms。[^16] 托管 API 的 p50 大约在 50 到 100 ms。[^17] 如果整个 RAG 预算有 800 ms，这还能接受；如果只有 200 ms，那就跳过重排序、认下召回上的损失，或者换一个更小的蒸馏模型。

## 用 LLM 做重排序的诱惑

还有第三种方案不断被提起：跳过交叉编码器，把候选直接发给 GPT-4o 或 Claude，用提示词让它给一个排好序的列表。ZeroEntropy 2025 年 7 月的基准把这个权衡算成了具体数字：列表式（listwise）LLM 重排序得 0.78 NDCG@10，交叉编码器是 0.74——真实但有限的 5% 提升，代价是 9 倍成本、35 倍延迟。[^12]

这道算术几乎把所有生产场景都淘汰了。一个每天做 1 万次重排序的编程助手，账单会从交叉编码器的 30 美元变成 LLM 的 270 美元，每个查询新增的延迟从 60 ms 变成 2 秒。LLM 列表式重排序只留给极高价值查询上最后那次 5 进 1（医学文献检索、法律取证）——只有在那种地方，边际质量提升才值这笔账。其余所有场合，用交叉编码器。

## 2026 年的默认技术栈

在 Elasticsearch、Pinecone、Weaviate 以及我见过的大多数生产团队那里，已经收敛成型的架构是这样的：

1. **BM25** 跑倒排索引，返回前 50 到 100 个。
2. **稠密检索**跑 HNSW 索引，返回前 50 到 100 个。
3. 用 `k=60` 做 **RRF 融合**，产出 25 到 50 篇文档的候选列表。
4. **交叉编码器重排序**处理这份候选列表，返回前 5 到 10 个。
5. **LLM** 拿到这 5 到 10 段重排后的文本作为上下文。

> [!WARNING]
> 最常见的错误是两路检索深度不对称。如果向量检索返回 200 个候选而 BM25 只返回 20 个，稠密列表就凭数量压过 RRF，你的精确匹配查询会悄无声息地退化。两边取同样的 `k`；如果你的流量偏向精确 token 查询（商品编码、人名、标识符），就让 BM25 多取一些。

让这一切可调试的做法，是把每个查询在每个阶段的候选都记下来：BM25 返回了哪些文档、稠密检索器返回了哪些、融合后的顺序是什么、重排序又把它改成了什么。客户说搜索漏掉了一个显而易见的结果时，你能精确还原是哪一级把它丢了。没有这份日志，你只能猜是编码器、融合还是重排序出了问题——而答案通常是你没怀疑的那一个。

放到架构尺度上，[RAG 流水线](../../hld/part-9-ai-ml-system-design/01-rag-pipelines.md)和[大规模向量检索](../../hld/part-9-ai-ml-system-design/02-vector-search-at-scale.md)讲的是白板视角：这套栈如何在分片、协调节点和每秒承载数千次查询的重排序集群之间分布。

---

## 参考文献

[^1]: "BM25, Vector and Reranking Reference 2026," digitalapplied.com, May 2026. https://www.digitalapplied.com/blog/hybrid-search-bm25-vector-reranking-reference-2026
[^2]: "Okapi BM25," Wikipedia, accessed June 2026. https://en.wikipedia.org/wiki/Okapi_BM25
[^3]: "BM25 Okapi Ranking Calculator," metricgate.com, April 2026. https://metricgate.com/docs/bm25-okapi-ranking/
[^4]: Rengifo, J., "How to measure and improve Elasticsearch search recall: from 0.43 to 0.75 with hybrid search," Elasticsearch Labs, May 4, 2026. https://www.elastic.co/search-labs/blog/elasticsearch-relevance-tuning-improve-recall
[^5]: Bailis, P., "Hybrid search revisited: introducing the linear retriever in Elasticsearch!" Elasticsearch Labs, May 28, 2025. https://www.elastic.co/search-labs/blog/linear-retriever-hybrid-search
[^6]: Cormack, G.V., Clarke, C.L.A., Buttcher, S., "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods," Proc. 32nd ACM SIGIR, 2009. https://dl.acm.org/doi/10.1145/1571941.1572114
[^7]: Elasticsearch Reference, "Reciprocal Rank Fusion (RRF) retriever," Elastic, accessed June 2026. https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html
[^8]: Mark, "Hybrid Retrieval Architecture Best Practices for Production AI Systems [2026]," Markaicode, May 2026. https://markaicode.com/architecture/hybrid-retrieval-architecture-best-practices-2026/
[^9]: NirantK / Superlinked, "Find the best retrieval strategy for your RAG," Superlinked docs, 2025. https://superlinked.com/docs/examples/benchmark
[^10]: Jacob, M., Lindgren, E., Zaharia, M., Carbin, M., Khattab, O., Drozdov, A., "Drowning in Documents: Consequences of Scaling Reranker Inference," arXiv:2411.11767, November 2024. https://arxiv.org/abs/2411.11767
[^11]: "Reranking and Cross-Encoders for RAG: BGE, Cohere, Jina (2026)," LocalAIMaster, May 2026. https://localaimaster.com/blog/reranking-cross-encoders-guide
[^12]: "Should You Use LLMs for Reranking? A Deep Dive into Pointwise, Listwise, and Cross-Encoders," ZeroEntropy Blog, July 2025. https://zeroentropy.dev/articles/should-you-use-llms-for-reranking-a-deep-dive-into-pointwise-listwise-and-cross-encoders/
[^13]: Muratbekova, U., "Best Reranker for RAG: We tested the top models," Agentset Blog, November 7, 2025. https://agentset.ai/blog/best-reranker
[^14]: Voyage AI, "Pricing," docs.voyageai.com, accessed June 2026. https://docs.voyageai.com/docs/pricing
[^15]: BAAI, "bge-reranker-v2-m3 model card," HuggingFace, March 2024. https://huggingface.co/BAAI/bge-reranker-v2-m3
[^16]: Mark, "PyTorch RAG Architecture: 4-Component Latency Budgets for Production," Markaicode, May 2026. https://markaicode.com/architecture/pytorch-rag-architecture/
[^17]: Mark, "Scalable Hybrid Retrieval Architecture: Production System Design for High Traffic [2026]," Markaicode, May 2026. https://markaicode.com/architecture/scalable-hybrid-retrieval-architecture/
