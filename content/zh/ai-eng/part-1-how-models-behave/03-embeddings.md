---
title: "嵌入向量（Embeddings）"
description: "什么是嵌入、为什么归一化向量上的余弦相似度与点积等价，以及建立在嵌入之上的四种工程模式：搜索、记忆、路由、聚类。"
slug: 03-embeddings
part: 1
chapter: 3
difficulty: beginner
prerequisites:
  - 01-tokens-and-context-windows
  - 00-the-model-landscape
tags: [llm, vector-search]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 嵌入向量（Embeddings）

把字符串 `"capital of France"` 发给 OpenAI 的 `/v1/embeddings` 端点，拿回来的是 1,536 个浮点数。发 `"Paris"`，拿回来的是另外 1,536 个数。算一下两者夹角的余弦，大约是 0.7。对 `"Paris"` 和 `"how to fix a leaky tap"` 做同样的计算，大约是 0.1。

0.7 和 0.1 之间的这段差距，就是嵌入（embedding）存在的理由。含义相近的两个字符串，生成的向量指向相近的方向；毫不相关的两个字符串，指向空间中相隔很远的地方。有了这一点，四个产品功能几乎不用额外代价就能拿到：懂改写的搜索、按语义召回的记忆、不需要训练分类器的路由，以及不需要标注的聚类。

本章讲的是心智模型和度量方式。深入的内容——挑向量库、切分文档、混合检索、重排序、agent 记忆——都在后面的部分。你在这里要弄清的是：嵌入到底*是什么*，该算哪个相似度数字，以及嵌入能撑起哪些东西的那张地图，这样后面几章落进来时，你已经有了承接它的框架。

## 向量不过是语义空间里的一个坐标

像制图师理解经纬度那样去理解嵌入。坐标相近的两个点，在实际位置上就彼此靠近。麻烦在于嵌入空间的坐标维度有几百上千个，而不是两个，而且你没法把它画出来。但规则是一样的：地图上近，就等于语义上近。

![浅白色平面上浮动着的三小簇点：左上角标着 "Paris" 和 "French city" 的两个点彼此靠近，中右部 "pizza" 和 "Italian food" 彼此靠近，底部中央是 "Python" 和 "code"，一个珊瑚色的查询菱形悬在 Python 那一簇附近](../../../ai-eng/part-1-how-models-behave/images/03-vector-space-proximity.png)
*含义相近的东西落在一起。查询落在哪个邻域，就匹配哪个邻域。*

这些数字本身不携带人能读懂的含义。第 47 维不是"法国程度"。单独看一个坐标，什么也说明不了。信息完全存在于向量之间的*关系*里，存在于任意两个向量离得多近里。契约只有这一条：语义相近，向量就相近。嵌入做的一切都建在这一个事实上。

网络是怎么学到这个的？靠对比训练（contrastive training）。训练时，模型看到意思相同的成对文本（一个问题与它的答案、两句改写），把它们的向量拉近就得到奖励；看到不相关的成对文本，把它们推远就得到奖励。在几十亿个文本对上重复下去，你最终得到一个函数，它把整个英语——通常还包括上百种其他语言——铺进一个用距离编码语义的几何结构里。[^1]

从模型的训练方式可以直接推出几条实用结论：

- **模型吃进文本，吐出固定长度的向量。** 不生成、不采样、没有 temperature。同样的输入每次都给你同样的向量（同一个模型快照内）。
- **输出维度由模型决定。** OpenAI 的 `text-embedding-3-small` 返回 1,536 个浮点数，`text-embedding-3-large` 返回 3,072 个。Cohere 的 `embed-v4` 返回 1,536 个，Google 的 `gemini-embedding-001` 最多 3,072 个。[^1][^2][^3]
- **输入有 token 上限。** OpenAI 的模型最多接受 8,192 个 token，Cohere `embed-v4` 接受 128,000 个，Google 的 `gemini-embedding-001` 接受 2,048 个。[^1][^2][^3] 更长的内容要么被静默截断，要么必须先切块，这正是[切分](../../../ai-eng/part-6-retrieval-augmented-generation/01-document-processing-and-chunking.md)要处理的失败模式之一。
- **不同模型的向量不可比。** 每个模型对空间的切法都不一样。拿一个 OpenAI 向量和一个 Cohere 向量算余弦相似度是没有意义的。一个索引只用一个模型，然后坚持下去。

最后一条比听起来更重要。升级嵌入模型时，你必须先把整个语料库重新嵌入一遍才能上线。把两个模型版本的向量混在同一个索引里，是让搜索系统悄悄返回胡说八道的最快方式。[^4]

## 余弦相似度、点积，以及让两者等价的技巧

你手上有两个向量，想要一个数字来说明它们有多像。实际可选的只有两种做法，而在你真正会用到的那些嵌入上，它们会塌缩成同一个运算。

**余弦相似度（cosine similarity）** 度量两个向量之间的夹角，取值从 -1（方向相反）到 1（方向相同）。它不关心两个向量各有多长。夹角相同的两个向量得分相同，不管一个短一个长。

**点积（dot product）** 就是原始的乘加求和：`a[0]*b[0] + a[1]*b[1] + ...`。它同时在意夹角*和*长度。一个夹角尚可的长向量，可以赢过一个夹角完美的短向量。

技巧在这里：如果两个向量都归一化成单位长度，也就是 `||a|| = ||b|| = 1`，点积就*等于*余弦相似度。长度被约掉，公式塌缩，少一次运算拿到同一个数。

```python
import numpy as np

def cosine_similarity(a, b):
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

a = np.array([0.2, 0.8, -0.1, 0.5])
b = np.array([0.3, 0.7,  0.0, 0.4])

# 把两个向量都归一化为单位长度。
a_unit = a / np.linalg.norm(a)
b_unit = b / np.linalg.norm(b)

print(f"cosine on raw vectors:        {cosine_similarity(a, b):.4f}")
print(f"dot on unit vectors:          {np.dot(a_unit, b_unit):.4f}  <- same number")
print(f"dot on raw vectors:           {np.dot(a, b):.4f}  <- magnitude leaks in")
# cosine on raw vectors:        0.9832
# dot on unit vectors:          0.9832  <- same number
# dot on raw vectors:           0.8200  <- magnitude leaks in
```

为什么这对你的代码重要？因为主流商业嵌入 API 默认返回的就是单位归一化的向量。OpenAI 说得很直接："OpenAI embeddings are normalized to length 1, which means that cosine similarity can be computed slightly faster using just a dot product."[^1] Cohere 和 Google 返回的也是归一化向量。所以实践上，余弦相似度和点积之间的选择消失了：你调用 `np.dot`，两个都一起算出来了。

那么决策规则很短：

- **默认：在单位归一化向量上算点积。** 它更快，是每个向量数据库都专门优化的度量，而且在商业嵌入上它和余弦完全等价。Postgres `pgvector`、Pinecone、Qdrant、Weaviate 都把它列为归一化输入下的推荐度量。
- **普通文本检索不必再犹豫。** 如果你用商业 API 或 sentence-transformers 模型嵌入文本，正在纠结该用哪个度量，答案就是这个。
- **唯一的例外是推荐系统。** 有些推荐模型，尤其是最大内积搜索那一类，是在*未归一化*向量上用点积目标训练出来的，其中长度本身是用来编码物品流行度的。这种情况下做归一化等于把信号扔掉，余弦会把长尾物品排错。Steck、Ekanadham 和 Kallus（Netflix，2024）证明，对某些带正则的矩阵分解嵌入，余弦相似度会产生"任意、因而无意义"的排序。[^5] 那篇论文和 Pinecone 的相似度指南给出的规则是一致的：度量的选择要匹配模型训练时优化的目标。[^5][^6]

对文本嵌入来说，这意味着余弦。对*你实际会调用*的文本嵌入来说，这意味着对 API 返回的东西调用 `np.dot`。

欧氏距离是某些库提供的第三种选择，但对单位归一化向量而言，它排出来的文档顺序和余弦完全一样，只是绝对数值不同。选一个，定下来，继续往下走。

把这件事从两个向量扩到整个语料库，只需要一行 numpy。把所有文档的嵌入堆成一个形状为 `(n_docs, dim)` 的矩阵，按行归一化，一次矩阵向量乘法就同时算出与每一篇文档的相似度分数：

```python
def top_k_similar(query, corpus, k=3):
    """corpus: (n_docs, dim) array. Returns (doc_index, score) pairs."""
    q = query / np.linalg.norm(query)
    corpus_unit = corpus / np.linalg.norm(corpus, axis=1, keepdims=True)
    scores = corpus_unit @ q                        # 每行做一次点积
    top = np.argsort(scores)[::-1][:k]
    return list(zip(top.tolist(), scores[top].tolist()))
```

这就是小规模语义搜索的全部引擎。文档量在几十万以下，一台笔记本上这个速度就够上线了。再往上，你要换成一个用近似最近邻索引的向量数据库（HNSW 是当下的默认选择），但核心的数学仍然是这个点积。向量库做的是完全相同的比较，只是聪明地跳过了大部分行。这一层算法，以及什么时候真的需要专用向量库、什么时候用你已有数据库上的 `pgvector` 就够了，都在[嵌入与向量检索](../../../ai-eng/part-6-retrieval-augmented-generation/02-embeddings-and-vector-search.md)里。

## 嵌入能撑起哪些能力

一旦你手上有了"文本相近，向量就相近"这个基本操作，四个产品功能就从中长出来。每一个在别处都有完整一章；这里给的是一张地图，好让你在某个问题出现时知道自己该伸手去拿哪一个。

![中央一个靛蓝色的方框标着 "Embedding"，四条箭头向外辐射到四个带标签的方框：上方的 "Search" 是珊瑚色，右侧 "Memory"，左侧 "Routing"，下方 "Clustering"](../../../ai-eng/part-1-how-models-behave/images/03-what-embeddings-power.png)
*一套向量表示，四种工程模式。搜索是最常见的入口。*

**搜索（Search）。** 把语料库里的每一篇文档离线嵌入一次，向量存进数据库。请求到来时，嵌入用户的查询，让数据库按余弦相似度找出最近的文档。它和 `WHERE body LIKE '%foo%'` 的区别在于："what time does the store close" 能匹配到标题为 "store hours" 的文档，一个字都不重合。这是 RAG 的机房重地，而怎么挑嵌入模型、怎么切分文档、什么时候混合检索胜过纯嵌入，都在[为什么需要检索](../../../ai-eng/part-6-retrieval-augmented-generation/00-when-you-actually-need-rag.md)及其后续几章里。

**记忆（Memory）。** 一个能跨会话记住事情的聊天 agent，通常不会把整段历史塞进下一个 prompt——上下文窗口会爆，成本也会爆。它做的是把每条消息或抽取出来的事实嵌入、存下向量，然后在每一轮检索出与当前查询最相关的少数几条。机制是搜索；*用途*是让 agent 记得"你上周提过你对花生过敏"，而不用重读整份记录。这部分在[记忆与状态](../../../ai-eng/part-5-context-engineering/03-memory-and-state.md)和[agent 记忆与任务状态](../../../ai-eng/part-7-agent-engineering/06-agent-memory-and-task-state.md)里有深入讨论。

**路由（Routing）。** 你收到一个请求，手上有三四个可选的系统提示词（一个处理账单问题，一个处理技术支持，一个处理闲聊）。把每条路由的简短描述各嵌入一次。请求到来时，嵌入用户的消息，选向量最近的那条路由。这就是一个零样本分类器，没有训练数据、没有标注、不用微调。作为第一道过滤，它效果好得出奇；碰上真正横跨两条路由的请求就会失效，这时候的升级方案是一个小分类器，或者让 LLM 当一次路由器。完整讨论在[模型路由](../../../ai-eng/part-7-agent-engineering/01-the-five-workflow-patterns.md)。

**聚类与去重。** 对 50,000 张支持工单的嵌入跑 k-means，你会得到"关于账单的工单""关于登录的工单""其实是夸我们、只是寄错地址的工单"这样的分组。OpenAI 自己的嵌入指南在 Amazon 评论上演示了这一点，不需要任何标注就恢复出了那些明显的簇（好评、差评、狗粮）。[^1] 去重是同一个想法最紧的版本：两篇文档的向量余弦相似度高于 0.95，几乎肯定是近似重复，包括精确匹配会漏掉的那些改写。

这四种用法的模式是同一个：同一个基本操作，做不同的活。你不需要为搜索和路由分别挑模型；挑一个好的通用嵌入模型，凡是合适的地方都用它。

## 怎么选模型，以及大多数人掉进去的坑

2026 年中的购物清单大致长这样：

| 模型 | 维度 | 上下文 | 价格（每 100 万输入 token） | 数据截至 |
|---|---|---|---|---|
| OpenAI `text-embedding-3-small` | 1,536 | 8,192 tok | $0.02 | 2026 年 6 月[^1] |
| OpenAI `text-embedding-3-large` | 3,072 | 8,192 tok | $0.13 | 2026 年 6 月[^1] |
| Cohere `embed-v4` | 1,536 | 128,000 tok | $0.12 | 2026 年 6 月[^2] |
| Google `gemini-embedding-001` | 最高 3,072 | 2,048 tok | 见 Vertex AI 定价 | 2026 年 6 月[^3] |
| BAAI `bge-m3`（开放权重） | 1,024 | 8,192 tok | 自托管 | 2026 年 6 月[^7] |

坑在排行榜上。Massive Text Embedding Benchmark（MTEB）是标准记分板，很容易把它当成一个单一的性能数字来读。OpenAI 报告 `text-embedding-3-small` 是 62.3%，`text-embedding-3-large` 是 64.6%。[^8] Cohere 报告 `embed-v4` 是 65.2%，不过这个数字来自 2025 年 4 月的一个二手来源（Vercel 的 AI Gateway 页面），而不是 Cohere 自己发布的基准，所以只当方向参考，别当权威结论。[^2]

MTEB 分数*不*告诉你的是：模型在*你的*数据上会表现如何。这个基准是 56 个任务的平均。总榜第一的模型，在法律文本、代码、你在意的某门非英语语言，或者你的用户真正写出来的那类改写上，可能排到第十。2024 年的一项研究发现，领域专用嵌入模型在领域任务上稳定胜过通用模型；差距大小不一，但方向在他们测试的每个领域上都成立。[^9]

这里的纪律，也正是[评测](../../../ai-eng/part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md)在别处一直讲的纪律：用真实用户查询配上已知的好文档，建一个小评测集，然后在*这个*上量召回率，而不是在 MTEB 上。两百个查询/文档对就够把两个模型区分开。分数接近就选便宜的那个；在你的数据上差距是真的，就选好的那个。

还有一个值得知道的旋钮是**维度缩减（dimension shortening）**。现代嵌入模型（`text-embedding-3-*`、`gemini-embedding-001`、`embed-v4`）用一种叫 Matryoshka Representation Learning 的技术训练，完整向量的前 256 或 512 维本身就是一个可用的、质量略低的嵌入。[^10] 你向 API 要 512 维，拿回一个 512 维向量，存储成本大约是默认 3,072 维的六分之一。OpenAI 公布的基准显示，`text-embedding-3-large` 缩到 256 维仍然胜过旧的 `text-embedding-ada-002` 的完整 1,536 维。[^8] 文档量到一百万时，省下的存储是实打实的（float32 下从 12 GB 降到 2 GB）；到一万时，不值得为它费脑子。

> [!WARNING]
> **把嵌入模型钉死在某个具体快照上，并给每个存下来的向量打上产出它的模型版本。** 托管嵌入端点在模型升级前后共用同一个 URL。厂商在你脚下换掉权重的那天，用新模型嵌入的查询就开始跟用旧模型嵌入的语料做比较，检索质量会以任何测试都抓不到的方式漂移。一篇 2026 年的实践者文章描述了这个故障最糟的版本：厂商侧的一次嵌入轮换，悄无声息地让一个季度的 A/B 测试结论全部失效，因为实验平台没有为那个唯一变动的变量留出字段。[^4] 请把嵌入模型升级当成数据库 schema 迁移来对待：蓝绿一套新索引，原子切换，并且拒绝在同一个存储里混用不同版本的向量。

嵌入是这本书里单位代码量承载分量最重的基本能力。一次 API 调用，几千个浮点数，你就把"给我找含义大致是这样的文档"从一道难题变成了一个矩阵乘法。从第 5 部分往后，各章都在不断兑现这件事，有时用于检索，有时用于记忆，有时用于 agent 内部的路由决策。机制没有变。变的是你向这个几何结构提出的问题，以及你索引得有多聪明，好让你能在规模上把问题问出来。

## 参考文献

[^1]: OpenAI, "Vector embeddings" (Embeddings guide), retrieved June 2026. https://platform.openai.com/docs/guides/embeddings
[^2]: Vercel AI Gateway, "Embed v4.0 by Cohere, Specs, Pricing & API," retrieved June 2026 (Cohere `embed-v4` MTEB 65.2% figure dated April 2025; single secondary source). https://vercel.com/ai-gateway/models/embed-v4.0/about
[^3]: Google Cloud, "Text embeddings API" (Vertex AI generative AI), page dated June 10, 2026. https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/text
[^4]: Tian Pan, "The Embedding Model Rotation That Shadowed Your A/B Test for a Quarter," tianpan.co, June 2, 2026. https://tianpan.co/blog/2026-06-02-the-embedding-model-rotation-that-shadowed-your-ab-test
[^5]: Steck, H., Ekanadham, C., & Kallus, N. (Netflix), "Is Cosine-Similarity of Embeddings Really About Similarity?", arXiv:2403.05440, March 8, 2024. https://arxiv.org/abs/2403.05440
[^6]: Pinecone, "Vector Similarity Explained," Pinecone Learning, January 2024. https://www.pinecone.io/learn/vector-similarity/
[^7]: BAAI / FlagEmbedding, "BGE-M3," HuggingFace model card, January 30, 2024; paper arXiv:2402.03216. https://huggingface.co/BAAI/bge-m3
[^8]: OpenAI, "New embedding models and API updates," OpenAI blog, January 25, 2024. https://openai.com/index/new-embedding-models-and-api-updates/
[^9]: "Do We Need Domain-Specific Embedding Models? An Empirical Investigation," arXiv:2409.18511, September 2024. https://arxiv.org/abs/2409.18511
[^10]: Kusupati, A. et al., "Matryoshka Representation Learning," arXiv:2205.13147, NeurIPS 2022. https://arxiv.org/abs/2205.13147
