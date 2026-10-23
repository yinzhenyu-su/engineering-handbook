---
title: "评估检索"
description: "RAG 评测主要是检索评测。用大白话讲清 recall@k 与 MRR、如何从真实查询构建评测集，以及把检索失败与生成失败区分开的 2×2 矩阵。"
slug: 07-evaluating-retrieval
part: 6
chapter: 7
difficulty: intermediate
prerequisites:
  - 00-when-you-actually-need-rag
  - 03-hybrid-search-and-reranking
  - part-4-eval-fundamentals/02-your-first-eval-set
tags: [llm, rag, search]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 评估检索

Amazon AWS AI 在 2024 年做了一次对照实验，覆盖十个领域、4,162 条查询。他们固定生成器，把检索器从 BM25 换成 E5-Mistral；结果在他们测过的每个生成器上，端到端 F1 都跳升了 5 到 10 个点。然后他们反过来做：固定检索，替换生成器。收益更小，而且不稳定。[^1]

这就是 RAG 质量的形状，它应该改变你把评测力气花在哪里的方式。生成器只能忠于检索器端上来的东西。如果正确的那个块没进上下文窗口，再好的提示工程、再大的模型、再长的思维链都救不了这个回答。所以实践结论很直白：**大部分 RAG 评测就是检索评测，在你为生成打分花掉第一块钱之前，先把检索修好**。

好消息是检索评测属于便宜的那一类。Recall@k 和 MRR 是确定性的、跑得快，而且不需要 LLM 裁判。贵的部分（忠实度、上下文召回）在后面，它们存在的意义主要是抓检索指标看不见的生成侧失败。顺序很重要。

## Recall@k 告诉你答案是否在场

拿一条本该由语料里两篇文档回答的查询：`doc1` 和 `doc2`。你的检索器返回五个块：`[doc3, doc1, doc7, doc2, doc5]`。Recall@5 是相关文档出现在前五名任意位置的比例。两篇金标文档都在里面，所以 Recall@5 = 1.0。把窗口砍到三个，只有 `doc1` 进来了；Recall@3 = 0.5。

这就是这个指标的全部。Recall@k 不管位置，排第 1 和排第 5 得分一样。它在意的是*覆盖率*：不管你交给生成器的是什么，支撑证据有没有进名单？

```python
def recall_at_k(retrieved_ids: list, relevant_ids: set, k: int) -> float:
    """Fraction of relevant docs found in top-k results."""
    top_k = retrieved_ids[:k]
    hits = sum(1 for doc_id in top_k if doc_id in relevant_ids)
    return hits / len(relevant_ids) if relevant_ids else 0.0


retrieved = ["doc3", "doc1", "doc7", "doc2", "doc5"]
relevant = {"doc1", "doc2"}
print(recall_at_k(retrieved, relevant, k=3))  # 0.5
print(recall_at_k(retrieved, relevant, k=5))  # 1.0
```

正确的 `k` 就是你生产系统用的那个 `k`。如果流水线往 LLM 里塞五个块，Recall@5 就是直接对应容量决策的那个指标。2026 年 3 月的 StratRAG 基准把这点说得很明确：他们的混合检索器（BM25 加 MiniLM）Recall@5 = 0.905，作者称其为“一个实用的上界：给生成器喂前 5 篇检索文档，大约 90% 的问题它都能拿到完整的支撑证据。”[^2] 高过这个天花板，再做生成侧的工作也没用。低于它，你测的就是模型能不能替一个漏掉证据的检索器补锅。

## MRR 告诉你它排到了多高

Recall@k 对顺序无感。如果你在意最重要的那个块是落在第 1 位而不是第 5 位，你要的是 MRR（mean reciprocal rank，平均倒数排名）。它是所有查询中第一篇相关文档 `1/rank` 的平均值。首命中在第 1 名得 1.0，第 2 名得 0.5，第 5 名得 0.2，没命中得 0。

```python
def mrr(retrieved_ids: list, relevant_ids: set) -> float:
    """Reciprocal rank of the first relevant doc."""
    for rank, doc_id in enumerate(retrieved_ids, start=1):
        if doc_id in relevant_ids:
            return 1.0 / rank
    return 0.0
```

MRR 存在的原因是“lost in the middle”（中间信息丢失）。LLM 对长上下文里最前和最后的位置注意力最强，中间的东西被低估。[^3] 一篇金标文档埋在 5 个里的第 4 位，技术上说检索成功了，但生成器可能没看见。当你加一个重排序器时，会动的指标是 MRR；Recall@k 可能一步不动。

两个一起看：

- **Recall@k 高、MRR 高：** 检索器找得到对的文档，并且放在顶部。健康。
- **Recall@k 高、MRR 低：** 文档找到了，但排得靠后。加重排序器。
- **Recall@k 低：** 文档压根没找到。这时 MRR 还没意义，先修召回。

还有一个你会见到的指标 nDCG@k（归一化折损累计增益），它通过支持分级相关性（一篇文档可以部分相关或完全相关，不只是是/否）把前两个指标都泛化了。当你有真带分级的人工标注标签时用它。对二值的合成评测集——也就是大多数团队手上的东西——Recall@k 和 MRR 用更低的成本装下了同样的信息。[^4]

## 在拿到标注之前先搭好评测集

上面这些指标需要（查询，金标文档）对。大多数团队一对都没有。手工标注语料是显然错误的答案：慢、贵，而且产出一个跟不上生产的冻结快照。真正管用的捷径是反向生成。

从语料里挑一个块出来。让 LLM 写一个答案完全落在这个块里的问题。这个块成了金标文档，生成的问题成了查询，于是你得到一行带标注的数据，全程不用人参与。[^5]

```python
def build_eval_row(chunk: str, llm_generate) -> dict:
    """Reverse-generate a query from a document chunk."""
    q_prompt = (
        "Read the passage and write a single factual question "
        "whose answer is entirely contained in it. "
        "Return only the question.\n\n"
        f"Passage: {chunk}"
    )
    question = llm_generate(q_prompt).strip()
    a_prompt = (
        f"Answer this question using only the passage.\n"
        f"Question: {question}\nPassage: {chunk}"
    )
    return {
        "user_input": question,
        "gold_chunk_id": chunk_id_for(chunk),
        "reference": llm_generate(a_prompt).strip(),
    }
```

几百条这样的数据就能让你脱离零起点。StratRAG 基准用了 200 条验证样本，并报告在这个量级上检索系统之间的排名已经稳定。[^2] 你不需要 4,000 行数据才知道混合检索在你的语料上打不打得过 BM25。

合成查询有个已知弱点：LLM 默认生成事实型问题（“X 是哪年成立的？”），因为这类问题好生成。真实用户问的东西是模糊的、多跳的、半成型的。所以一旦有了任何生产流量，就去挖它。Hamel Husain 推荐的评测集从合成起步，然后朝生产日志漂移，就像回归测试集朝真实 bug 漂移一样。[^5]

有三条规则值得守住：

- **抽样拿过负反馈的查询**（点踩、改写重问、中途放弃）。这些行最可能是检索坏掉的地方。
- **把无法回答的查询放进去。** 如果一个问题在语料里没有支撑，正确行为是“我不知道”。只装可回答问题的集合会把这个失败模式整个藏起来；Barnett 等人把它列为失败点 FP1。[^6]
- **先聚类再标注。** 按主题和类型给查询分组，然后在每个簇内抽样。这能防止评测集过度代表上周二正流行的那类东西。

关于从生产流量养大评测集这套更完整的方法论，[你的第一套评估集](../part-4-eval-fundamentals/02-your-first-eval-set.md) 把整个循环讲得很细。检索这边的特殊之处是：你的“金标标签”是一个文档 ID，而不是对答案的通过/不通过判定。

## 忠实度检查答案是否对上上下文，而不是对上世界

检索器正常之后，下一个失败模式是生成器编东西。忠实度（faithfulness）衡量答案里的每一条断言是否被检索到的块所蕴含。关键在于，它*不*检查答案在现实中是否正确，而是检查答案有没有扎根于检索器带回来的东西。

RAGAS 的实现用 LLM 把答案拆成一条条原子陈述，然后（同样用 LLM）逐条问这个陈述能不能从上下文推出来。得分是通过的比例。在 WikiEval 数据集上，这个方法和人工判断的一致率达到 0.95，远高于 GPT-score 基线的 0.72。[^7]

忠实度不需要参考答案。你不用有 ground-truth 答案，只需要检索到的上下文和生成的回答。这让它便宜到可以在每条生产 trace 上跑，也是大多数团队最先上它的原因。

陷阱在这里。忠实度是一个*条件*指标。它问的是“给定这段上下文，答案有根据吗？”它问不了“这段上下文本身好不好？”如果检索器带回了错误的文档，而生成器忠实地总结了那篇错文档，忠实度得 1.0，用户拿到的是自信满满的错误答案。

缓解办法是把忠实度和**上下文召回**（context recall）一起跟踪，后者把参考答案拆成陈述，逐条对照检索到的上下文检查。上下文召回需要参考答案（忠实度不需要），但这两个合起来才给你真正的诊断力。单看一个分数会说谎。

## 检索失败还是生成失败：2×2 矩阵

每个错误答案从外面看都一样。用户问了东西，系统回了话，回话很烂。“正确的文档从来没被检索到”和“文档检索到了但模型无视了它”，这两者的修法完全不同。如果你分不清，就会花几周去优化错误的那一半。

诊断工具是落在上下文召回与忠实度上的一个 2×2：

![一张二元网格图，横纵轴分别是上下文召回与忠实度，每个象限标出对应的失败模式与应采取的动作](../../../ai-eng/part-6-retrieval-augmented-generation/images/07-retrieval-vs-generation-failure.png)
*四个象限把一个错误答案路由到产生它的那个组件。*

把它当路由表来读：

- **召回高、忠实度高。** 两个组件都在正常工作。可以发布。
- **召回高、忠实度低。** 检索器找到了证据，模型无视或扭曲了它。这是生成失败：检查“lost in the middle”效应，收紧提示里“只准用上下文内容作答”的指令，或者把 `k` 调小，让相关块不至于淹在噪声里。
- **召回低、忠实度高。** 这一格最危险。检索器带回了错误的材料，生成器忠实地总结了错误材料，而你的忠实度看板一片绿。去修检索器：切块策略、嵌入模型，或者从 BM25 换成混合检索。StratRAG 显示混合检索在多跳查询上把 Recall@5 从 0.815（单用 BM25）提到 0.905。[^2]
- **召回低、忠实度低。** 检索烂到生成器手上没东西可用。别费劲诊断生成了，先修检索再重测。

代码短到可以直接粘进 notebook：

```python
def diagnose(context_recall: float, faithfulness: float) -> str:
    """Route a wrong answer to its dominant failure mode."""
    low_recall = context_recall < 0.5
    low_faith = faithfulness < 0.7
    if low_recall and low_faith:
        return "both broken: fix retrieval first"
    if low_recall:
        return "retrieval failure: faithful to bad context"
    if low_faith:
        return "generation failure: context present, model ignored it"
    return "healthy"
```

那两个阈值（0.5、0.7）只是起点；拿你自己 trace 里一小份人工标注子集把它们校准一下。RagChecker 在 280 个实例、10 个领域上做的元评测显示，断言级召回与忠实度和人工判断的 Pearson 相关系数是 0.62，远高于任何单一的端到端指标。[^1] 2×2 不是完美的神谕，但它是最便宜、又能稳定把你路由到正确组件的诊断。

由此推出的排查顺序：当你坐下来改进一个 RAG 系统时，跑评测集，看这张矩阵，把时间花在失败聚集的地方。大多数团队会发现自己的失败挤在右下象限——这也是“RAG 评测主要是检索评测”这句话一直经得起现实检验的原因。

## 参考文献

[^1]: Ru, Dongyu et al., "RagChecker: A Fine-grained Framework for Diagnosing Retrieval-Augmented Generation," Amazon AWS AI, arXiv:2408.08067, August 2024. https://arxiv.org/abs/2408.08067
[^2]: Patodiya, Aryan, "StratRAG: A Multi-Hop Retrieval Evaluation Dataset for Retrieval-Augmented Generation Systems," California State University Fresno, arXiv:2604.22757, March 2026. https://arxiv.org/abs/2604.22757
[^3]: Liu, Nelson F. et al., "Lost in the Middle: How Language Models Use Long Contexts," arXiv:2307.03172, July 2023. https://arxiv.org/abs/2307.03172
[^4]: Yu, Hao et al., "Evaluation of Retrieval-Augmented Generation: A Survey," arXiv:2405.07437, May 2024. https://arxiv.org/abs/2405.07437
[^5]: Husain, Hamel, "Q: How should I approach evaluating my RAG system?", AI Evals FAQ, hamel.dev, July 2025. https://hamel.dev/blog/posts/evals-faq/how-should-i-approach-evaluating-my-rag-system.html
[^6]: Barnett, Scott et al., "Seven Failure Points When Engineering a Retrieval Augmented Generation System," Deakin University, arXiv:2401.05856, January 2024. https://arxiv.org/abs/2401.05856
[^7]: Es, Shahul et al., "Ragas: Automated Evaluation of Retrieval Augmented Generation," arXiv:2309.15217, September 2023. https://arxiv.org/abs/2309.15217
