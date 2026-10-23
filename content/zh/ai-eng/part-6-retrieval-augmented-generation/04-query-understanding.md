---
title: "查询理解"
description: "改写、多查询、问题分解与路由：弥合用户敲进去的措辞和检索器能匹配上的内容之间的落差。"
slug: 04-query-understanding
part: 6
chapter: 4
difficulty: intermediate
prerequisites:
  - 02-embeddings-and-vector-search
  - 03-hybrid-search-and-reranking
  - part-1-how-models-behave/03-embeddings
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 查询理解

用户敲进 `cheap flights europe summer`。真正能回答这条问题的文档写的是 `affordable air travel to European destinations between June and August`。两个字面重叠的词都没有。BM25 一无所获。稠密检索器表现好一些，但它仍然被要求跨越一道用户根本不知道存在的鸿沟：查询是三个关键词，文档是一段话，嵌入空间把它们当成两种不同的东西。

这就是词汇不匹配（vocabulary mismatch）问题，“RAG 在我们这儿就是不好用”这类抱怨大多源自这里。检索器没问题。chunk 没问题。用户的措辞对人来说也没问题。坏的是两者之间的那座桥，而且升级嵌入模型修不好它。

查询理解（query understanding）是一类技术：在查询进入检索器*之前*对它做改写、扩展、分解或路由，好让检索器看到能匹配上的东西。有四种值得掌握，大致按成本递增排序：改写（多一次 LLM 调用）、多查询（N 次并行调用）、分解（一串相互依赖的调用）、路由（在所有环节前面放一个分类器）。本章的任务是告诉你该先上哪一个、按什么顺序上，以及每一个要花你多少钱。

![左侧是一条只有三个词的用户查询，右侧是一段格式规整的长文档，两边下方各有一片互不重叠的词云，一条标着 embedding similarity 的弧线跨过两者之间的空隙，尽管没有任何共同词汇](../../../ai-eng/part-6-retrieval-augmented-generation/images/04-vocabulary-gap.png)
*检索鸿沟：真正能回答这条查询的文档，可能一个词都不与它共用。*

拿一个简单的预算来校准每种技术。每一次改写、每一个变体、每一个子问题，在用户看到任何东西之前都至少多一次 LLM 调用。小模型典型每次调用 200 到 500 ms，四次串行调用就在检索还没开始之前加了一秒延迟。每种技术要回答的问题都一样：在你的评测集上换来的召回提升，值不值得这些调用？

## 改写：多一次调用

最便宜的一招是在检索前把查询换个说法。一次 LLM 调用，一条提示词：*“把这个问句改写成一句适合文档检索的独立陈述句。”* 模型会展开缩写、消解代词、去掉对话里的填充语，产出的文本在句法上更贴近你的语料。微软的 Azure AI Search 把它做成了内置功能，并报告截至 2024 年 11 月在他们自己的基准上 NDCG@3 提升了 4 个点，收益最大的是打在基于词项的索引上的短小、含糊的查询。[^1]

你并不总是需要它。2025 年的一项研究发现，纯提示词改写可能在本来表述良好的查询上*降低*稠密检索质量：改写会把边缘词意译进查询，把嵌入推到检索器学到的簇之外。[^2]由此得出的规则是有条件的：查询是对话式的、含代词或指代，或者在你的评测集上检索精确率一直偏低时才改写。查询本身已经是干净利落的陈述句时就跳过。全局打开之前先做 A/B 测试。

更有意思的变体是 HyDE：假设文档嵌入（Hypothetical Document Embeddings）。[^3] 你不再嵌入用户的问题，而是让 LLM 针对它写一段简短的*答案*，再嵌入这段答案，用*它*的向量去检索。这个假设可以是部分幻觉出来的。这没关系，因为编码器会把它映射到嵌入空间里真实答案文档所在的区域附近。风格会迁移；内容在你真正对真实语料跑相似度搜索时会被筛掉。

```python
def hyde_retrieve(query: str, llm_generate, embed, vectorstore, top_k: int = 5):
    """
    HyDE: generate a plausible answer passage, embed THAT, retrieve real docs.
    Gao et al., arXiv:2212.10496 (Dec 2022).
    """
    # 假设里可能编造事实；我们从不会把它展示给用户。
    hypothesis = llm_generate(
        f"Write a concise passage that answers: {query}\n\nPassage:"
    )
    hyp_vec = embed(hypothesis)
    return vectorstore.similarity_search_by_vector(hyp_vec, k=top_k)
```

HyDE 适合的是技术语料上的零样本稠密检索，那种场景下用户查询短、文档长且满是领域词汇。它不适合的是生成假设的模型不懂你的领域。一条关于罕见病的医学查询，可能生成一段描述*另一种*同名症状疾病的假设，于是嵌入带着高置信度指向了错误的簇。检测办法：在你的评测集上把 HyDE 和直接查询检索并排跑，比 recall@k。HyDE 赢就上；不赢，说明你的模型在这个领域的编造能力还不够。

## 多查询：N 种改写，融合起来

改写只挑一种措辞。多查询（multi-query）一次挑好几种，并行跑过检索器，再把结果列表融合。赌的是召回：不同措辞能把单一措辞打不中的文档捞出来，因为每种措辞拉到的嵌入空间区域略有不同。

融合这一步用倒数排名融合（Reciprocal Rank Fusion）。对任一列表 `L` 中出现的每篇文档 `d`，得分是 `sum over lists of 1 / (k + rank_L(d))`，`k=60` 取自 RRF 原始论文。[^4]在多个列表里都排得靠前的文档得分最高；只在某个列表第 80 名露过一次的文档贡献几乎为零。RAG-Fusion 是这个模式的经典成文，而 RRF 得分正是让这套技术对噪声变体宽容的原因：糟糕的改写，其结果自己会沉到底部。[^4]

```python
def reciprocal_rank_fusion(ranked_lists: list[list[str]], k: int = 60) -> list[str]:
    """RRF: score(d) = sum_L 1 / (k + rank_L(d)). k=60 is standard."""
    scores: dict[str, float] = {}
    for ranked in ranked_lists:
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=lambda d: scores[d], reverse=True)


def multi_query(question: str, llm_generate, retriever, n: int = 3):
    """Generate n phrasings, retrieve each, fuse with RRF."""
    raw = llm_generate(
        f"Generate {n} different phrasings of this question, one per line, "
        f"no numbering.\nQuestion: {question}"
    )
    variants = [q.strip() for q in raw.strip().split("\n") if q.strip()][:n]

    ranked, doc_map = [], {}
    for v in variants:
        results = retriever.get_relevant_documents(v)
        ids = [doc.metadata.get("id", doc.page_content[:40]) for doc in results]
        for doc, did in zip(results, ids):
            doc_map[did] = doc
        ranked.append(ids)

    return [doc_map[i] for i in reciprocal_rank_fusion(ranked) if i in doc_map]
```

三个到五个变体是实用区间。少于三个，不如只改写一次、只付一次调用。多于五个，你付的是递减的召回收益，加进去的噪声比信号还多。检索器调用应该并发跑；五个变体还串行多查询，等于把五个检索往返叠进用户的延迟预算里。

值得盯的失败模式是主题漂移。变体生成器对“留在用户意图之内”没有硬约束，有时会把一条含糊查询解读到某个方向上去，捞回主题相邻但无关的文档。Rackauckas 在英飞凌的产品信息系统上评估 RAG-Fusion 时，把这列为主要的破坏来源。[^4]修法是加一道相关性闸门：把生成的每个变体嵌入，算它与原查询的余弦相似度，低于约 0.7 的变体直接丢掉。代价是每个变体多一次嵌入调用，能把最糟的漂移当场拦住。

## 分解：一条查询扛不住整个问题时

多查询帮的是“某种措辞打不中正确文档”。当问题真的需要两份分处不同文档的事实，它就帮不上了。*“2024 年 TIOBE 排名第一的编程语言，做它的那家公司的 CEO 是谁？”* 你得先找到这门语言，再找到这门语言背后的公司，再找到这家公司的 CEO。原问题无论怎么改写，都没法一次把三份文档都捞回来，因为第二次、第三次查找依赖你还没有的答案。

这是桥接型问题（bridge question），它要的是分解：把原问题拆成有序的子问题，按顺序逐个检索并回答，再把每个答案喂进下一个子问题的检索。IRCoT 把这件事形式化为交替进行思维链推理与检索；每一句推理就成为下一次检索的查询。[^5]在 HotpotQA、2WikiMultihopQA、MuSiQue 和 IIRC 上，IRCoT 相对单次 RAG 基线把检索最多提升了 21 个点、下游问答最多提升了 15 个点，并且从 GPT-3 泛化到 Flan-T5-large 无需重训。

```python
def sequential_decompose(question: str, llm, retriever, max_hops: int = 3):
    """
    Bridge questions: each sub-question's answer informs the next.
    Cap at 3 hops in production; latency scales linearly.
    """
    context, answer = [], None
    for hop in range(max_hops):
        prompt = (
            f"Question: {question}\n"
            f"Known so far: {context}\n"
            "If you can answer now, output FINAL: <answer>. "
            "Otherwise output NEXT: <one sub-question to retrieve>."
        )
        step = llm(prompt).strip()
        if step.startswith("FINAL:"):
            return step[6:].strip(), context
        sub_q = step[5:].strip() if step.startswith("NEXT:") else step
        docs = retriever.get_relevant_documents(sub_q)
        context.append({"sub_q": sub_q, "docs": [d.page_content for d in docs]})
    return llm(f"Question: {question}\nContext: {context}\nAnswer:"), context
```

这里有两个要紧的形状。第一，`max_hops` 在生产环境封顶三跳，没有例外。每一跳是一次检索往返加一次生成调用；在 MuSiQue 的难题上跑四到八跳，用户在第一个 token 流回来之前已经等了五秒。三跳能覆盖大多数真实的桥接问题，也让你的 p99 延迟可控。第二，当子问题 2 依赖子问题 1 的答案时，顺序分解无法并行，而这个依赖关系正是你要分解的全部理由。如果子问题是*相互独立*的（比如聚合型问题，“把这五款产品比一比”），并行分解会并发发起再合并结果，但那是另一个问题。

坑在于对不需要分解的东西做分解。*“法国的首都是哪？”* 语法简单，事实原子。把它丢进分解器，要么生成一个退化的子问题（原问题重复一遍），要么更糟：生成两个冗余子问题，结果互相矛盾，把生成器绕晕。给分解器加一道复杂度闸门：一个在多跳与单跳样本上训过的轻量二分类器，或者对桥接型标记做正则（英文里的 *“that… which… whose…”* 这类连接词，配合多个实体）。不是桥接型问题，就检索一次然后收工。

## 路由：别用一套策略打所有查询

上面三种技术是在同一条检索流水线上做质量优化。路由（routing）是上游的闸门，决定*哪条*流水线先跑起来。前提是：不同查询类型有不同的最佳检索器。词法搜索（机制见[混合检索与重排](./03-hybrid-search-and-reranking.md)）在精确的产品编号、人名和冷僻技术术语上更强。稠密向量搜索在换种说法的表达和语义相似上更强。混合检索在意图混杂时更强。把所有查询都送进同一条流水线，等于让每种查询类型时不时拿到次优结果。

规则式路由是正确的起点。把查询分到少数几个类型，查表得到对应的流水线：

```python
from enum import Enum

class QueryType(Enum):
    FACTOID  = "factoid"   # 精确实体/日期  -> 词法
    SEMANTIC = "semantic"  # 概念相似      -> 稠密
    HYBRID   = "hybrid"    # 意图混杂      -> 词法 + 稠密 + RRF
    MULTIHOP = "multihop"  # 桥接型问题    -> 先分解

ROUTING_RULES = {
    QueryType.FACTOID:  "lexical",
    QueryType.SEMANTIC: "dense",
    QueryType.HYBRID:   "hybrid",
    QueryType.MULTIHOP: "decompose_then_retrieve",
}

def classify_query(query: str, llm_classify) -> QueryType:
    label = llm_classify(
        "Classify into one of: factoid, semantic, hybrid, multihop.\n"
        f"Query: {query}\nType:"
    ).strip().lower()
    try:
        return QueryType(label)
    except ValueError:
        return QueryType.HYBRID  # 含糊情况下的安全默认
```

SelRoute 是 2025 年针对这个模式做的一次评估，场景是长期对话记忆检索：用 bge-base-en-v1.5 加规则驱动路由，在 LongMemEval_M 上 Recall@5 达到 0.800。[^6]教训是：对于从查询到检索流水线的路由，简单规则就够用，因为查询类型与检索器的对应关系建立在众所周知的属性上。学习式路由要打败规则，得等你的查询分布大到、怪到规则开始漏判。

兜底和规则一样重要。像*“BERT-K 里的 K 代表什么？”*这样的查询，可能被分成事实型（它确实是在问一个精确词项），但答案不在任何词法索引里；它首先要的是“BERT-K 是什么”的语义理解。低置信度的分类一律默认混合检索。盯住路由分布；兜底率爬过约 20%，说明你的规则集缺了一个类别。

另一种路由问题出现在多模型 RAG 里：手上有好几个可用的 LLM，要在*已经拿到检索文档*的前提下挑出最适合这条查询的那个。同一条查询，某个模型不检索就能答对，加上检索反而答错（噪声把它带偏），另一个模型则恰好相反。RAGRouter 用一个约 136M 参数的对比式交叉编码器处理这件事；截至 2025 年 5 月，它在五个知识密集型任务上平均比最好的单个带 RAG 的模型高 3.61%，在单张 RTX 4090D 上每条查询耗时 0.011 秒。[^7]在你需要跨多个模型供应商路由、且成本与质量的差异大到值得多付这点延迟之前，你不需要它。

在架构尺度上，查询路由不再是质量优化，而变成一个基础设施组件：在向量库、SQL 数仓和知识图谱前面放一个语义路由服务，配上缓存、熔断器和按租户的策略。HLD 手册的[企业级 RAG 架构](../../hld/part-9-ai-ml-system-design/01-rag-pipelines.md)一章讲了白板上该怎么画。

## 合起来用：预算法则

这四种技术可以组合。真实的生产流水线先路由查询，再按路由结果有条件地改写或分解，然后可选地在选定的流水线内部扇出到多查询。陷阱是全默认打开，然后在 p99 上发现检索还没开始，你已经串起了四次 LLM 调用。

| 技术 | 什么时候默认开 | 什么时候跳过 | 成本 |
|---|---|---|---|
| **改写** | 查询是对话式的，或含代词 | 查询已经是干净利落的陈述句 | +1 次 LLM 调用 |
| **HyDE** | 零样本稠密检索、技术语料 | LLM 不懂这个领域（会编错） | +1 次 LLM 调用、+1 次嵌入 |
| **多查询** | 在你的评测集上单查询召回偏低 | 延迟吃紧；查询本身已经很具体 | +N 次 LLM 调用（并行） |
| **分解** | 桥接型问题：两个以上相互依赖的事实 | 单跳事实型；跳数封顶 3 | +N 次串行调用 |
| **路由** | 你有不止一条检索流水线 | 你反正只跑一个检索器 | +1 个小分类器 |

> [!WARNING]
> 延迟累积是杀死生产环境里查询理解的那个失败模式。天真地把改写 + 多查询（5 个变体）+ 分解（3 跳）+ 路由分类器叠起来，检索上游就有十次 LLM 调用。就算每次 300 ms，那也是取回第一篇文档前的 3 秒。独立的调用要并行，昂贵的技术要用便宜的分类器把关，转换用小号微调模型，重复查询的转换结果要缓存。如果你的端到端 p99 预算是两秒，你负担得起其中一种技术，也许两种，四种全上不行。

有一个反论值得承认。对于小规模、静态、能舒舒服服装进长上下文窗口的语料，把整套文档塞进提示词、让模型自己做检索，可能强过切块 RAG；2025 年的一次评估发现，在中等语料规模下，长上下文在基于维基百科的问答上普遍优于 RAG。[^8]问题在成本形状：长上下文按查询付费，无限期地付下去；带查询理解的 RAG 建索引时付一次，之后每次查询只付零头。到 2026 年中，实践者的共识是：小规模静态语料、延迟和成本都不敏感时，长上下文赢；其余一切场景，带查询理解的检索赢。

让这一切可度量的纪律是评测集。上面每种技术都有一个能在同一套固定查询集上算出来的 recall@k 数字，“这玩意儿有没有用”是个数字，不是感觉。把任何一种接进去之后，下一步是[评估检索](../../../ai-eng/part-6-retrieval-augmented-generation/07-evaluating-retrieval.md)。

## 参考文献

[^1]: Microsoft Azure AI Foundry Blog, "Raising the Bar for RAG Excellence: Query Rewriting and New Semantic Ranker," November 2024. https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/raising-the-bar-for-rag-excellence-query-rewriting-and-new-semantic-ranker/4302729
[^2]: "When Prompt-Only LLM Refinement Helps and Hurts Dense Retrieval," arXiv:2603.13301, 2025. https://arxiv.org/abs/2603.13301
[^3]: Luyu Gao, Xueguang Ma, Jimmy Lin, Jamie Callan, "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE), arXiv:2212.10496, December 2022. https://arxiv.org/abs/2212.10496
[^4]: Zackary Rackauckas, "RAG-Fusion: a New Take on Retrieval-Augmented Generation," IJNLC Vol. 13 No. 1, February 2024, arXiv:2402.03367. https://arxiv.org/abs/2402.03367
[^5]: Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, Ashish Sabharwal, "Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive Multi-Step Questions" (IRCoT), ACL 2023, arXiv:2212.10509. https://arxiv.org/abs/2212.10509
[^6]: "Query-Type-Aware Routing for Long-Term Conversational Memory Retrieval" (SelRoute), arXiv:2604.02431, April 2025. https://arxiv.org/abs/2604.02431
[^7]: Jiarui Zhang et al., "Query Routing for Retrieval-Augmented Language Models" (RAGRouter), arXiv:2505.23052, May 2025. https://arxiv.org/abs/2505.23052
[^8]: "Long Context vs. RAG for LLMs," arXiv:2501.01880, 2025. https://arxiv.org/abs/2501.01880
