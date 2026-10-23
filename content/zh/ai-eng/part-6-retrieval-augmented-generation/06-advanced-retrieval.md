---
title: "高级检索"
description: "三种在特定查询上胜出原生 RAG 的模式：智能体式检索（agentic retrieval）、GraphRAG、视觉原生 PDF 检索。各自何时值得那份复杂度，何时不值得。"
slug: 06-advanced-retrieval
part: 6
chapter: 6
difficulty: advanced
prerequisites:
  - 00-when-you-actually-need-rag
  - 02-embeddings-and-vector-search
  - 03-hybrid-search-and-reranking
  - part-2-working-with-model-apis/03-function-and-tool-calling
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 高级检索

有三种模式被推销成原生 RAG（vanilla RAG）的升级版，而对上正确的查询形态，这三种都确实有用。但它们也足够贵——默认上它们，你会在质量毫无提升的情况下真金白银地多花钱。

先看几个头条数字，全部来自 2025 年 6 月同一份针对 100 万 token 语料的基准测试。原生 RAG：每次查询约 880 token。GraphRAG 全局检索：每次查询约 331,000 token。这是 375 倍。[^1] 在同一语料上做简单的事实检索，GraphRAG 局部检索的准确率是 49.3%，而带重排序器（reranker）的普通 RAG 是 60.9%。[^2] 为问题挑错了工具，你可以多付 40 倍的钱，换回一个更差的答案。

这一章要说实话：每一种模式都是为某一种特定查询形态才值得那份复杂度。除此之外的地方，更简单的技术栈都赢。重点不是学会三个新玩具，而是看清自己正站在哪个岔路口。

![一棵四分支的决策树，按查询类型说明每种检索模式何时适用，每条分支上标着成本量级](../../../ai-eng/part-6-retrieval-augmented-generation/images/06-advanced-retrieval-decision-tree.png)
*岔路口。大多数查询走最左边那条分支，然后到此为止。*

## Agentic RAG：把检索做成模型调用的工具

标准 RAG 里，流水线先检索、模型后生成，顺序是固定的；检索有没有用、要不要再试一次，模型没有发言权。

Agentic RAG（智能体式检索）把它反过来。检索变成一个以工具形式暴露给模型的函数，暴露方式跟[函数调用与工具调用](../part-2-working-with-model-apis/03-function-and-tool-calling.md)里的任何其他工具一样。模型自己决定什么时候调、用什么查询、以及根据返回结果决定要不要再调。控制循环沿用 ReAct（Reason+Act）模式：思考、行动、观察，重复到得出答案或者用完预算。[^3]

```python
MAX_STEPS = 8  # 硬上限，不是软告警

def run_agentic_rag(user_query: str) -> str:
    messages = [{"role": "user", "content": user_query}]
    for step in range(MAX_STEPS):
        response = llm_call(messages)
        if response["type"] == "answer":
            return response["content"]
        tool_result = retrieve(response["query"])
        messages.append({"role": "tool", "content": tool_result})
    raise RuntimeError(f"Agent exceeded {MAX_STEPS} steps")
```

值得这份复杂度的形态是多跳（multi-hop）问题。“我们 EMEA 地区的客户里，哪些在上一季度续了约，又在最近 30 天提过一张标记为紧急的工单？”这需要两次检索，第二次依赖第一次的结果，再加一次模型在脑子里完成的连接。单趟 RAG 做不到。模型必须先发一个查询、读结果，再决定下一个问什么。

代价是真实的。Agentic RAG 消耗的 token 是单趟 RAG 的 3~10 倍，延迟是 2~5 倍（单一从业者来源，截至 2026 年 4 月）。[^4] 在 FAQ 机器人或单点事实查询上，这份开销换不来东西：模型“决定”要检索是早就注定的事，循环只是多绕几趟。

有两条生产规则不能让。第一条是步数上限。没有硬顶的循环就是一颗成本炸弹：历史累积到 8,000 token 时，一次工具调用失败，在模型写出第一个输出 token 之前就已经烧掉 8,000 个输入 token。在不稳定的工具上重试十次就是 80,000。[^5] LinkedIn 工程团队报告过，在加上治理之前，智能体跑一次的成本是单轮调用的 62~136 倍。[^6] 把 MAX_STEPS 设在 4~8，超了就抛错；绝不要悄悄返回一个残缺答案。

第二条是延迟叠加。一次 200ms 的工具调用单看没问题。一个会话里七次，就意味着模型想完之后用户还要再等 4.5 秒，因为每次调用都要重付一遍初始化开销。[^7] 给循环做预算要按秒算，不是按步算。

## GraphRAG：用知识图谱回答全局性问题

原生 RAG 取回与查询相似的文本块。答案落在一个块里时，这办法好用；答案需要理解整个语料时，它就不行了。

“我们一万份客户访谈里的主要主题是什么？”是标准例子。没有任何一个块包含答案。Top-k 相似度检索回来十份访谈，内容是查询碰巧在向量空间里靠近什么就是什么，做出来的归纳很浅。

微软研究院的 GraphRAG 用一个两段式索引解决这个问题。[^8] 让 LLM 读遍每个块，抽取命名实体和实体之间的关系，建起一张知识图谱。然后用 Leiden 算法把图划分成实体连接稠密的社区（community），再由 LLM 为每个社区在多个层级上写一段自然语言摘要。查询分两种模式：

- **局部检索（local search）：** 把查询向量化，找出 top-k 实体，拉取它们的描述、关系和来源块。在 100 万 token 的语料上，每次查询约 39,000 token。[^1]
- **全局检索（global search）：** 对每份社区摘要做 map-reduce，每批生成一个部分答案，再归并。同一语料上每次查询约 331,000 token。[^1]

```python
def graphrag_query(query: str) -> str:
    if is_global_question(query):
        # 对所有社区摘要做 map-reduce（约 331K token）
        return global_search(query)
    # 实体 + 关系 + 块上下文（约 39K token）
    return local_search(query)
```

在需要全局理解的查询上，GraphRAG 确实赢。Edge 等人在播客和新闻数据集上报告，由 LLM 评判，它在全面性和多样性上对朴素 RAG 的胜率是 70~80%。[^9] 在这类查询上，它的开销还只有分层原文摘要的 2~3%。要“把整个数据集总结一遍”，它就是对的工具。

其余场合就难看了。独立的 GraphRAG-Bench 研究（Xiang 等人，2025 年 6 月）发现，在事实检索上 GraphRAG 局部检索的准确率是 49.3%，而带重排序器的 RAG 是 60.9%。在事实型查询上，全局检索的上下文相关性掉到 9.4%，带重排序器的 RAG 是 77.8%。[^2] Han 等人佐证了同样的形态：在 Natural Questions 上准确率低 13.4%，在时效性查询上低 16.6%。[^10] 如果你的用户问的是“谁、什么、何时、何地”，GraphRAG 只加成本、不加准确率，还经常倒退。

第二个实在的问题是索引开销。完整版 GraphRAG 用 LLM 把每个块读两遍（一遍抽取，一遍做社区摘要）。微软自己后续那篇也承认这点：明确的动机就是“避开可能高得无法承受的前置索引成本”，这才有了 **LazyGraphRAG**（2024 年 11 月）。[^11] LazyGraphRAG 用 NLP 名词短语抽取替掉 LLM 抽取，索引成本降到完整版 GraphRAG 的 0.1%（与向量 RAG 同量级），而在 500 次测试的查询期相关性预算下，它只花完整版全局查询成本的 4%，却在局部和全局查询上都优于其他所有方法。[^11]

默认规则：事实查询用原生 RAG。当主导查询是全局性的（“比较各方观点”“有哪些模式”“影响是什么”），语料足够稳定、索引能摊到几千次查询上，而且你已经量过、LLM 评判的全面性正是你在意的指标，这时候才去动 GraphRAG。除非你确实需要那张知识图谱作为可共享的产物，否则用 LazyGraphRAG 而不是原版。截至 2025 年，只有不到 15% 的企业在生产环境用了图检索，这大体上等于真正以全局理解为主要查询形态的语料占比。[^12]

## 视觉原生检索：ColPali 与 OCR 流水线的终结

满是图表、信息图、多栏表格和扫描图的 PDF 会弄坏基于文本的 RAG。标准流水线（OCR -> 版面检测 -> 分块 -> 生成图注 -> 文本向量化）会摧毁承载含义的视觉结构。一张柱状图变成“Q1 Q2 Q3 Q4 12 18 24 19”，没有坐标轴也没有图例，向量（embedding）完全不知道自己刚看的是什么。

ColPali（Faysse 等人，2024 年 6 月；v5 为 2025 年 2 月）绕开了整套文本抽取栈。[^13] 它把每页 PDF 渲染成图片，送进视觉语言模型（vision-language model，原版用 PaliGemma-3B，当前最优配置用 ColQwen2.5-7b），为每个图像 patch 存一个 128 维向量。448px 分辨率下是 1,024 个 patch 加 6 个指令 token，也就是每页 1,030 个向量。

打分用**后期交互（late interaction）**配 MaxSim 算子。查询被切成 token，每个查询 token 有自己的向量；对每个查询 token，找出它与页面上任一 patch 的最大点积，再对所有查询 token 求和。数学就是一行：

```python
import torch

def max_sim(query_vectors: torch.Tensor, page_vectors: torch.Tensor) -> float:
    """query_vectors: [n_query_tokens, 128]; page_vectors: [1030, 128]."""
    sim = torch.einsum("qd,pd->qp", query_vectors, page_vectors)
    return sim.max(dim=1).values.sum().item()
```

在视觉复杂的文档上，收益很大。ViDoRe 基准上，ColPali 在 10 个任务上平均 81.3 nDCG@5。TAT-QA（带表格的财报）上，ColPali 得 65.8，对 OCR 后的文本跑 BM25 是 44.0。信息图任务上，文本基线根本没法评，因为 OCR 返回的东西没法用。[^13] 索引延迟在 NVIDIA L4 上是每页 0.39 秒，而完整 Unstructured + 图注流水线在最高质量档要 7.22 秒。截至 2025 年 2 月，ColQwen2.5-7b-multilingual 在公开 ViDoRe 榜上排第 1。[^14]

代价在存储。float16 下，一页要占 257.5 KB 向量数据，而单向量稠密表示约 8.6 KB。[^13] 100 万页的语料上，这就是 244 GB 与 8.2 GB 的差别。把 ColPali 天真地铺到 5,000 万份扫描件上，你会先耗尽磁盘，而不是先耗尽用户。

两个生产手段能让它变得可行。**Token 池化（token pooling）** 取 3 倍因子，向量数减少 66.7%，准确率保留 97.8%。[^13] **二值量化（binary quantization）** 把每个 128 维向量压进 128 bit、用 int8 存储，每页成本降到约 16 KB（32 倍缩减），同时把 MaxSim 换成汉明距离，算力提速 3.5 倍。Vespa 演示了一条分阶段流水线（先用二值做候选检索，再对头部命中做 float 重排），在 DocVQA 上 nDCG@5 从 52.4 掉到 51.6，用 0.8 个点换存储上的收益。[^15]

另一个生产陷阱是数据传输。把候选页向量拉到外部排序器的天真做法，每次查询要搬约 32 MB（20 个查询 token × 2,000 个候选页 × 1,030 个向量 × 16 字节）。只要吞吐量像样一点，网络会先于 GPU 饱和。像 Vespa 那样把 MaxSim 和索引放在一起，或者把计算推到存储层。[^15]

语料真的是视觉型时用 ColPali：图表、信息图、表格、多栏版式、OCR 会丢掉结构的扫描件。规模化纯文本语料别用，30 倍的存储税换不来任何东西。只有 CPU、没有 GPU 做编码的基础设施也别用。ColPali 作者自己提到，以文本为主的文档“在所有评测领域上也由 ColPali 检索得更好”[^13]；这话没错，但如果你存的是 5,000 万页法律合同，它就跟你的场景无关。

## 到底该怎么选

| 模式 | 什么时候值得 | 相对原生 RAG 的成本 | 什么时候跳过 |
|---|---|---|---|
| **Agentic RAG** | 多跳问题；模糊查询；有多个检索后端要选 | 3~10 倍 token，2~5 倍延迟 | 单点事实查询、FAQ 机器人 |
| **GraphRAG** | 稳定语料上的全局理解（“主题”“模式”） | 约 375 倍 token（全局），完整索引要拿 LLM 把语料过两遍 | 事实型查询；零散或稀疏文本；要求新鲜度的语料 |
| **ColPali** | 图表、表格、信息图、扫描图密集的 PDF | 每页约 30 倍存储（二值量化后约 2 倍） | 纯文本语料；只有 CPU 的基础设施；未做量化且超过 1,000 万页的语料 |

采纳任何一个之前要问的问题，不是“它是不是更强”，而是“我的查询里，真正属于这个模式目标形态的占多大比例？”低于 20%，就把这些查询路由到高级模式，其余全留在便宜的路上。低于 5%，先别建。

[检索效果评估](./07-evaluating-retrieval.md)那一章讲怎么衡量升级有没有真的推动你的指标；[生产环境的 RAG](./08-production-rag.md)那一章讲索引成本的摊销与重建索引的代价——这两件事决定了 GraphRAG 的索引在你的流量下能不能自己回本。

## 参考文献

[^1]: Xiang, Zhishang et al., Figures 8-9 token count tables, "When to use Graphs in RAG: A Comprehensive Analysis for Graph Retrieval-Augmented Generation," arXiv:2506.05690, June 2025. https://arxiv.org/abs/2506.05690
[^2]: Xiang, Zhishang et al., Tables 2-3, "When to use Graphs in RAG," arXiv:2506.05690, June 2025. https://arxiv.org/abs/2506.05690
[^3]: Yao, Shunyu et al., "ReAct: Synergizing Reasoning and Acting in Language Models," ICLR 2023. https://arxiv.org/abs/2210.03629
[^4]: MarsDevs, "Agentic RAG: The 2026 Production Guide," marsdevs.com, April 2026 (single source). https://www.marsdevs.com/guides/agentic-rag-2026-guide
[^5]: tianpan.co, "Why Every Failed Tool Call Burns Your Token Budget," April 2026. https://tianpan.co/blog/2026-04-10-retry-storm-problem-agentic-systems
[^6]: Bhalsod, "2026 ROI (Policy RAG, Autonomous w/ Approvals)," LinkedIn Pulse, 2025-2026 (single source). https://www.linkedin.com/pulse/2026-roi-policy-rag-autonomous-w-approvals-fix-runaway-bhalsod-pus1f
[^7]: tianpan.co, "How Tool-Server Overhead Compounds by Agent Step 7," May 2026. https://tianpan.co/blog/2026-05-10-mcp-tool-server-cold-start-tax-latency-compounds-step-7
[^8]: Edge, Darren et al., "From Local to Global: A Graph RAG Approach to Query-Focused Summarization," arXiv:2404.16130, April 2024. https://arxiv.org/abs/2404.16130
[^9]: Edge, Darren et al., "GraphRAG: New tool for complex data discovery now on GitHub," Microsoft Research Blog, July 2, 2024. https://www.microsoft.com/en-us/research/blog/graphrag-new-tool-for-complex-data-discovery-now-on-github/
[^10]: Han, Haoyu et al., "RAG vs. GraphRAG: A Systematic Evaluation and Key Insights," arXiv:2502.11371, February 2025. https://arxiv.org/abs/2502.11371
[^11]: Edge, Darren, Ha Trinh, Jonathan Larson, "LazyGraphRAG: Setting a new standard for quality and cost," Microsoft Research Blog, November 25, 2024. https://www.microsoft.com/en-us/research/blog/lazygraphrag-setting-a-new-standard-for-quality-and-cost/
[^12]: tianpan.co, "The Architecture Decision Teams Make Too Late," April 2026 (single source). https://tianpan.co/blog/2026-04-19-graphrag-vs-vector-rag-architecture-decision
[^13]: Faysse, Manuel et al., "ColPali: Efficient Document Retrieval with Vision Language Models," arXiv:2407.01449v5, February 2025. https://arxiv.org/abs/2407.01449
[^14]: ViDoRe Leaderboard, Hugging Face Spaces, accessed February 2025. https://huggingface.co/spaces/vidore/vidore-leaderboard
[^15]: Bergum, Jo Kristian, "Scaling ColPali to billions of PDFs with Vespa," Vespa Blog, September 20, 2024. https://blog.vespa.ai/scaling-colpali-to-billions/
