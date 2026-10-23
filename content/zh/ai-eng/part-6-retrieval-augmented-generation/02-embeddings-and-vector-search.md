---
title: "向量嵌入与向量检索"
description: "怎么用自己的数据挑嵌入模型，为什么 pgvector 是默认的向量存储，HNSW 和 IVFFlat 到底在做什么，以及专用向量数据库真正值回票价的那条诚实阈值线在哪。"
slug: 02-embeddings-and-vector-search
part: 6
chapter: 2
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/03-embeddings
  - part-4-eval-fundamentals/02-your-first-eval-set
  - 01-document-processing-and-chunking
tags: [llm, rag, vector-search]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 向量嵌入与向量检索

你已经知道向量嵌入（embedding）是什么了。本章讲紧接着的四个决策，顺序就是大多数团队实际撞上它们的顺序：用哪个模型做嵌入、向量放在哪、在向量上建哪种索引，以及什么时候该停止用 PostgreSQL、花钱上专用方案。

这四个决策里有三个有强默认答案。第四个——模型选型——才是团队稳定搞错的那个，因为他们倚重的证据（MTEB 排名）衡量的并不是它需要衡量的东西。所以从它开始讲。

## 别照着排行榜挑嵌入模型

大规模文本嵌入基准（Massive Text Embedding Benchmark，MTEB）用 56 个公开任务给嵌入模型排名：维基百科检索、新闻分类、法律文本相似度，还有几十个同类任务。[^1] 截至 2026 年初，Google 的 Gemini Embedding 001 领跑英文榜，阿里的 Qwen3-Embedding-8B 领跑多语言榜，Voyage-3-large 是最强的托管 API 选项。[^1]

这些排名没有一条能告诉你，模型在你的 Salesforce 商机备注、你的药品试验摘要、你的内部工程工单上表现如何。MTEB 是在公开语料上取平均；整体排第八的模型，只要它的预训练数据分布恰好更贴合你的文本，就能在你这个领域上打赢榜首那个模型。[^1] 还有个污染的尾巴：若干排名靠前的 MTEB 模型已被记录到在训练中部分接触过基准测试集，所以即便公开排名本身也有一部分虚高。[^2]

解法是同一套纪律，就是[你的第一个评测集](../part-4-eval-fundamentals/02-your-first-eval-set.md)为提示词主张过的那套，搬到检索上：

1. 从你真实的语料里抽 500 到 2,000 篇有代表性的文档。
2. 写或收集 100 到 200 条真实用户查询，并标注已知的相关文档。（一个下午的活；这个集合以后每次比模型都能复用。）
3. 用每个候选模型分别嵌入查询和文档，跑最近邻检索，算 Recall@5 和 MRR@10。
4. 差距在五个点以内就选便宜的那个。不止五个点就选好的那个。

整个流程就这些。它每次都赢过读排行榜，而且一旦你有了这个评测集，你还顺带有了一个预警系统，用来发现某天托管服务商在你背后悄悄换了模型权重。

2026 年年中，这份评测一份合理的起步候选清单：

- **`text-embedding-3-small`，512 维**（2026 年 6 月每百万 token $0.02[^3]）。便宜，支持俄罗斯套娃截断，是大多数团队该先测的默认选项。
- **`text-embedding-3-large`**（每百万 token $0.13[^4]）。small 输了五个 Recall@5 点以上时用它，或者语料是多语言的时候（在 MIRACL 上 large 比 ada-002 高 23 个点[^4]）。
- **`voyage-3-large`**（每百万 token $0.18[^5]）。代码为主或专业领域的英文语料，它在这些地方已知表现好。
- **`bge-m3` 或 `arctic-embed-2.0` 自托管**。月用量越过约 500M token、API 的账算不过来时用，或者厂商换模型版本的风险在合同上不可接受时用。

维度这个旋钮值得用。`text-embedding-3-*` 两个系列和好几个开源模型都是用俄罗斯套娃表示学习（Matryoshka Representation Learning）训的，意思是你可以让 API 给 256 或 512 维，拿到一个质量略低但能用的向量，而不是默认的 1,536 或 3,072。[^6] OpenAI 自己的数据显示，`text-embedding-3-large` 截到 256 维，在 MTEB 上仍然打赢满尺寸 1,536 维的 `ada-002`。[^4] 存储省掉三分之二，索引明显更快，白送。

```python
# OpenAI：在嵌入时直接要更短的向量
from openai import OpenAI
client = OpenAI()

resp = client.embeddings.create(
    model="text-embedding-3-small",
    input=["the cat sat on the mat"],
    dimensions=512,  # 俄罗斯套娃截断；默认会是 1536
)
vec = resp.data[0].embedding  # 长度 512
```

在假设自己需要完整维度之前，先用截断后的维度跑一遍评测。大多数语料根本察觉不出差别。

## pgvector 就是默认的向量存储

对新的 RAG 负载来说，正确的向量库是一张装了 `pgvector` 扩展的 PostgreSQL 表。不是 Pinecone，不是 Qdrant，不是 Weaviate。是 PostgreSQL。

理由在运维层面，不在意识形态。你几乎肯定已经在跑一个 Postgres 实例。`CREATE EXTENSION vector` 会加一个 `vector(N)` 列类型和四个相似度运算符（`<=>` 余弦、`<->` L2、`<#>` 内积、`<+>` L1），它们能直接塞进普通 SQL 里用。[^7] 文档正文、元数据、嵌入向量、租户 ID 全在同一行。`WHERE tenant_id = $1` 这种过滤第一天就能用，不需要单独的“payload 索引”功能。你现有的备份、复制、可观测性免费就把向量存储覆盖了。不用再部署、监控、管版本第二个数据库，也不用付 AWS 出网费去跟它通信。

配置就四行：

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id        bigserial PRIMARY KEY,
    tenant_id text NOT NULL,
    content   text NOT NULL,
    embedding vector(512)
);

CREATE INDEX ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

查询就是在距离运算符上带 `ORDER BY` 的普通 `SELECT`：

```sql
SELECT id, content, embedding <=> $1 AS distance
FROM documents
WHERE tenant_id = $2
ORDER BY distance
LIMIT 10;
```

整个接口就这些。没有 SDK、没有 auth token、没有另一套重试策略。

“pgvector 只是玩具”这种条件反射已经过时好几年了。2025 年 5 月，Timescale 在单台 AWS r6id.4xlarge（按需 $835/月，128 GB 内存）上，用开放的 `ann-benchmarks` 套件跑了 5000 万个 768 维 Cohere 向量。装了 pgvectorscale 扩展的 pgvector 在 99% 召回率下达到每秒 471 次查询；Qdrant 在同一台机器、同样召回率下是 41 QPS。[^8] Postgres 吞吐量赢了 11 倍。Qdrant 赢了 P99 长尾延迟（99% 召回下 39 ms 对 75 ms），建索引只花了三分之一的时间。两个数字都是真的。但两个都不说明“你必须迁移”。

pgvector 做不到的事：

- **原生的横向分片。** Postgres 靠换更大的机器扩展，不靠加节点。只读副本能帮读，它们不分片索引。
- **直接索引 3,072 维向量。** `vector` 类型的 HNSW 和 IVFFlat 索引上限是 2,000 维。要用满尺寸的 `text-embedding-3-large`，你要么在嵌入时截断，要么存成 `halfvec(3072)`（FP16，索引可到 4,000 维），要么二值量化到 `bit` 列。[^9]
- **快速建超大索引。** 上面那个 5000 万的索引，在 pgvectorscale 的 StreamingDiskANN 上建了 11 个小时，在 Qdrant 上是 3.3 小时。[^8] 如果你每周都要重建索引，这个差距要命。

向量数不到 1000 万时，这些都不是致命问题。阈值我们稍后回来说。

## HNSW 与 IVFFlat 到底在做什么

在一百万个向量上做顺序扫描很慢。AWS 那份 pgvector 基准测试在 58,000 个 1,536 维向量上，顺序扫描每次查询 650 ms；加了 HNSW 索引后，同一个查询 1.5 ms。[^10] 这 400 倍的差距就是近似最近邻（approximate nearest neighbor，ANN）索引换来的东西，代价是掉一点点召回率。

pgvector 自带两种 ANN 索引。它们的工作方式差别很大，而二者之间的选择是 RAG 里少数有真正共识答案的决策之一。

**HNSW（Hierarchical Navigable Small World，分层可导航小世界）** 建一张多层图。每个向量成为一个节点。每个节点连到最近的若干邻居，顶层撒几条长距离边，底层是稠密的局部连接。要找一个查询的最近邻，你从顶层出发，贪心地朝离查询更近的那个邻居走，走不动了就下沉一层，重复。顶层负责把你快速带过整个空间，底层负责精确定位答案。

![一张三层的层次化图：顶层是稀疏的长距离边，中层更密，底层连接非常紧密。一颗珊瑚色的查询星标逐层下降，沿一条粗体路径从顶层的稀疏入口走到最底层的绿色目标节点。](../../../ai-eng/part-6-retrieval-augmented-generation/images/02-hnsw-layered-graph.png)
*HNSW 几跳就从稀疏的顶层走到稠密的底层。查询根本不会访问图里的大部分节点。*

三个参数，默认值都合理：

- **`m`**（默认 16）：每个节点的边数。越大图越密、召回越好、内存越多。生产上取值区间是 12 到 48。
- **`ef_construction`**（默认 64）：插入时构建器多努力去找好邻居。至少要 `2*m`。
- **`hnsw.ef_search`**（默认 40）：查询时检索多努力。用 `SET hnsw.ef_search = 100;` 按会话设置。召回低于目标就调高它。

HNSW 有两个性质对生产很重要。第一，你可以在空表上把索引建好，之后它永远能干净地吸收插入。第二，默认参数下召回对延迟的曲线非常好：大多数负载开箱就有 95% 以上的召回。[^11]

**IVFFlat** 是更老、更简单的设计。建索引时它对现有向量跑 k-means，把空间切成 `lists` 个 Voronoi 单元，每个单元锚在一个质心上。查询时它算查询到每个质心的距离，挑出最近的 `probes` 个单元，只扫这些单元里的向量。

两个参数：

- **`lists`**：单元数。pgvector 建议一百万行以内取 `rows/1000`，超过就用 `sqrt(rows)`。[^11]
- **`ivfflat.probes`**（默认 1）：查询时扫几个单元。建议起始值 `lists/10`（超过一百万时用 `sqrt(lists)`）。

IVFFlat 有一个坑，踩过的团队比任何别的 pgvector 陷阱都多。**索引必须在表里有代表性数据之后建。** 如果你在建表时就创建 IVFFlat 索引（大多数 ORM 和迁移工具的默认做法），k-means 在接近零行的数据上跑，所有质心都落在差不多同一个位置，查询返回垃圾，而且不报任何错。[^12] Supabase 的文档说得很直白："Building an IVFFlat index on an empty collection will result in significantly reduced recall."[^12]

逐项对比：

| | HNSW | IVFFlat |
|---|---|---|
| 空表上建索引 | 可以 | 不行，会静默坏掉 |
| 内存 | 多 2 到 5 倍 | 更少 |
| 建索引时间 | 更慢 | 更快 |
| 95% 召回下的查询延迟 | 更低 | 更高 |
| 处理插入 | 可以，无需重建 | 会漂移，建议重建 |
| 长期召回 | 稳定 | 随数据漂移而退化 |

决策规则很短：**新东西一律默认 HNSW。** 只有数据集基本静态、内存吃紧、且初始建索引时间比稳态查询速度更重要时，才用 IVFFlat。文献里对这点没有活的争论；HNSW 几年前就在在线负载的权衡对比中赢了，差距至今没缩小。

还有一个 0.8.0 的特性值得知道：HNSW 现在支持迭代扫描（`SET hnsw.iterative_scan = ON`），它会一直在图里走下去，直到有足够多的结果通过你的 `WHERE` 过滤。[^13] 在这之前，多租户表上一个选择性很高的过滤条件能把结果列表饿死，悄悄把召回打崩。如果你用的是多租户 pgvector，把它打开。

## 什么时候才真的需要专用向量数据库

大多数团队在第一周就选了 Pinecone 或 Qdrant，那时他们还没有任何测量出来的理由。成本是真的，而且是经常性的：又一套基础设施，自己的 API key、出网账单、备份流程、监控方案和值班轮换。收益则取决于他们当时还没有的负载特征。

基于公开基准和一线报告，老实画出来的阈值图：

- **向量数不到 100 万：** pgvector 配 HNSW，几乎任何实例规格。不用调。
- **100 万到 1000 万：** pgvector 配 HNSW，实例要大到能把索引装进 `shared_buffers`。对着你的评测集调 `ef_search`。
- **1000 万到 5000 万：** 还是 pgvector，但你会想要 pgvectorscale 的 StreamingDiskANN 和一个像样的实例（Timescale 那个 5000 万的基准用了 128 GB 内存）。[^8] 落地前先用评测集验证。
- **超过 5000 万，且 p99 有硬性的 10ms 以内 SLO：** 认真评估 Qdrant 或 Milvus。它们在高召回下的长尾延迟优势是真的，也有文档。[^8]

2026 年 4 月一份真实生产 RAG 的实践报告正好落在这个位置：200 万个 chunk，每天 500 到 2,000 次查询，pgvector 除了默认值没做任何调优就守住了 200ms 的 p95 预算。作者同时测了 Pinecone、Turbopuffer 和 Qdrant；没有一个值得迁移。[^14]

真正能证明该搬家的触发条件：

1. **原生横向分片是硬需求。** 如果你的语料确实塞不进一台足够壮的机器，pgvector 的纵向扩展模型就走到头了。
2. **几百个并发查询下 p99 要在 10ms 以内**，这是产品 SLO，不是愿望。5000 万向量、99% 召回下 Qdrant 的 39ms p99 对 Postgres 的 75ms，就是交互式界面能感觉出来的那种差距。[^8]
3. **你重建索引很频繁，11 小时的建索引时间在卡发布。** Qdrant 重建同一个 5000 万索引花了 3.3 小时。[^8]
4. **这个应用是纯向量负载，没有关系连接，也没有事务需求。** 当你从 Postgres 其它能力里什么都得不到的时候，你还在为它们付钱。

如果这四条一条都不符合你，你就还没到阈值。跑评测，盯仪表盘，等一个实测数字逼你迁移，而不是等厂商的博客文章。

## 真正搞坏生产的决策：模型版本管理

pgvector 对 Pinecone 的争论抢走了所有注意力。真正造成静默检索故障的决策，是嵌入模型升级。

两个不同模型产出的向量不可比。它们活在不同的坐标系里；拿 v1 的查询去和 v2 的文档算余弦相似度，得到的是一个数据库会欣然返回的无意义数字。当一个托管服务商在同一个 endpoint 背后推了新版本，或者你的团队从 `text-embedding-ada-002` 升到 `text-embedding-3-small` 只重嵌了一部分语料，你手上就是一个混合版本的索引：最近动过的文档结果漂亮，其它的全是噪声。没有错误日志。用户比工程师先发现。[^1]

防止这件事的模式是基于别名的版本管理，数据库团队做 schema 迁移用了几十年的同一招：

```python
# 1. 新模型到位。并行建一个新列和一个新索引。
"""
ALTER TABLE documents ADD COLUMN embedding_v2 vector(512);
-- backfill embedding_v2 with the new model in a background job
CREATE INDEX docs_hnsw_v2_20260612
    ON documents USING hnsw (embedding_v2 vector_cosine_ops);
"""

# 2. 应用指向一个视图，而不是某一列。
"""
CREATE OR REPLACE VIEW docs_current AS
    SELECT id, content, embedding_v1 AS embedding FROM documents;
"""

# 3. v2 通过评测集之后，用一条 DDL 换掉视图。
"""
CREATE OR REPLACE VIEW docs_current AS
    SELECT id, content, embedding_v2 AS embedding FROM documents;
"""
# 回滚就是同一条 DDL 指回 v1。零停机，没有混合状态。
```

三条规则让这个模式在实践中成立：

- 每个存下来的向量都打上产出它的模型名和版本。`embedding_v1` 这种命名可以；旁边再加一个 `model_version text` 列更好。
- 绝不在活索引上原地更新模型。永远并行建、验证、切换。
- 按计划对活索引跑你的检索评测集。查询和它 top-K 结果之间的平均余弦相似度是你的先行指标：当它没经过任何发布就发生偏移，说明有服务商在你背后悄悄换了权重。

技术上的修复是半天工作量。组织上的修复更难：得有人负责嵌入模型的生命周期，按计划跑评测，读仪表盘。大多数被索引漂移坑到的团队，是因为没人担这个职责。[^1]

## 参考文献

[^1]: Tian Pan, "Embedding Models in Production: Selection, Versioning, and the Index Drift Problem," tianpan.co, 2026 年 4 月 9 日. https://tianpan.co/blog/2026-04-09-embedding-models-production-versioning-index-drift
[^2]: "Leaks and duplications in the MTEB leaderboard," GitHub issue, embeddings-benchmark/mteb #1036. https://github.com/embeddings-benchmark/mteb/issues/1036
[^3]: Costgoat, "OpenAI Embeddings Pricing," 访问于 2026 年 6 月. https://costgoat.com/pricing/openai-embeddings
[^4]: OpenAI, "New embedding models and API updates," 2024 年 1 月 25 日. https://openai.com/index/new-embedding-models-and-api-updates/
[^5]: Voyage AI 定价，来自 MongoDB 博客与 Future AGI 的价目条目，访问于 2026 年 6 月。
[^6]: Kusupati et al., "Matryoshka Representation Learning," arXiv:2205.13147, NeurIPS 2022. https://arxiv.org/abs/2205.13147
[^7]: pgvector 项目，README 与运算符参考，访问于 2026 年 6 月. https://github.com/pgvector/pgvector
[^8]: Timescale, "Pgvector vs. Qdrant: Open Source Vector Database Comparison," 2025 年 5 月 5 日. https://medium.com/timescale/pgvector-vs-qdrant-open-source-vector-database-comparison-f40e59825ae5
[^9]: PostgreSQL News, "pgvector 0.7.0 Released," 2024 年 4 月. https://www.postgresql.org/about/news/pgvector-070-released-2852/
[^10]: AWS Database Blog, "Optimize generative AI applications with pgvector indexing: a deep dive into IVFFlat and HNSW," 2024 年 3 月. https://aws.amazon.com/blogs/database/optimize-generative-ai-applications-with-pgvector-indexing-a-deep-dive-into-ivfflat-and-hnsw-techniques/
[^11]: Neon, "Optimize pgvector search," Neon 文档，访问于 2026 年 6 月. https://neon.tech/docs/ai/ai-vector-search-optimization
[^12]: Supabase, "IVFFlat Indexes," Supabase 文档，访问于 2026 年 6 月. https://supabase.com/docs/guides/ai/python/indexes
[^13]: PostgreSQL News, "pgvector 0.8.0 Released," 2024 年 11 月. https://www.postgresql.org/about/news/pgvector-0.8.0-released-2952/
[^14]: alexcloudstar, "Vector Database Comparison 2026," 2026 年 4 月. https://alexcloudstar.com/blog/vector-database-comparison-2026/
