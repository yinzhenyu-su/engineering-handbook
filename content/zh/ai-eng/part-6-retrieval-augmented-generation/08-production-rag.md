---
title: "生产级 RAG"
description: "把 RAG 原型变成可运维系统的五个运维关注点：增量索引、重建索引税、多租户隔离、元数据过滤，以及分阶段延迟预算。"
slug: 08-production-rag
part: 6
chapter: 8
difficulty: advanced
prerequisites:
  - 02-embeddings-and-vector-search
  - 03-hybrid-search-and-reranking
  - 07-evaluating-retrieval
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 生产级 RAG

你的 RAG 昨天还答得对，今天就自相矛盾。表面上什么都没变——只是嵌入（embedding）服务商趁夜里悄悄上线了一次模型更新，而你的索引现在成了两个互不兼容的向量空间拼成的缝合怪。[^1]

这一课只有在生产环境里才学得到，原型阶段永远学不到。RAG 索引不是装好跑一次的流水线（pipeline），而是一个有状态的数据库：它会漂移、会堆积孤儿记录、会跨租户泄漏，上游任何东西一动它就悄悄退化。把 demo 变成真正能运维的系统，中间隔着五件事：文档变化时让索引保持最新、扛得住一次嵌入模型升级、安全地隔离租户、在不毁掉召回的前提下过滤元数据、以及给每个阶段分配延迟预算。这五件事互相牵连。任何一件上选错默认值，其余几件都会把它放大。

## 增量索引：先算哈希再嵌入

朴素的索引流水线每跑一次就把所有文档重新嵌入一遍。200 篇文档的 demo 上无所谓，1000 万（10M）分块（chunk）的语料上就是在烧钱——你在付费让嵌入 API 反复做同一份工作。修法只有一行便宜的代码：先对分块归一化后的文本算哈希（hash），只有哈希变了才调 API。

```python
import hashlib
import re

def compute_chunk_hash(text: str) -> str:
    """Normalize and hash a chunk for change detection."""
    normalized = re.sub(r'\s+', ' ', text.strip().lower())
    return hashlib.sha256(normalized.encode()).hexdigest()

def should_reembed(chunk_id: str, new_hash: str, hash_store: dict) -> bool:
    return hash_store.get(chunk_id) != new_hash
```

一定先归一化再算哈希，否则 CMS 保存时带上一个行尾空白，整篇文档就会被重新嵌入一遍。有了这道闸门，典型的企业流水线每次只重处理 10% 到 15% 的内容，而不是 100%，因为大多数文档在两次运行之间根本没改。[^2] 哈希可以存在按分块 ID 做键的 Redis 旁路（sidecar）里，也可以作为元数据直接挂在向量记录上。挂在向量上是少运维一个系统；放 Redis 则是让向量负载保持轻量。

三种模式，其中一个是默认值：

- **实时（事件驱动）。** CMS 或代码仓库在变更时发 webhook，流水线只处理那一篇文档。新鲜度（freshness）接近零延迟，代价是你得给数据源埋点。留给秒级就重要的内容用（实时定价、值班手册）。
- **微批（每 5 到 15 分钟）。** 默认选项。对着存储做哈希差分，处理增量，提交。爆炸半径可控，成本可预测。
- **定时批（每小时或每晚）。** 同样的哈希逻辑，更大的窗口。适合内部工具——用户能接受几小时的陈旧，换取批量 API 折扣。

规则是：先选微批；只有当产品真的依赖实时性时才升级到实时。

没人提醒你的那个故障模式是**孤儿堆积（orphan accumulation）**。文档被删除或重新分块后，旧向量会一直留在索引里，除非你主动清扫。几个月下来孤儿占比慢慢爬升，HNSW 图里长出指向空处的节点，p95 检索延迟在毫无明显原因的情况下往上漂。每次流水线跑完，都算一遍 `orphan_ids = indexed_ids - current_chunk_ids` 并分批删除。先用 `dry_run=True` 跑一次，确认集合大小符合预期再真删。

## 重建索引税：为什么升级嵌入模型并不免费

每个嵌入模型都定义了自己的坐标系。`text-embedding-3-large` 出来的向量和 `text-embedding-ada-002` 出来的向量不可比，哪怕名义维度相同——两个模型是用完全不同的几何结构来组织各自的高维空间的。把它们混在一个索引里不会报错，只会返回胡说八道。你一半语料在 A 空间，查询落在 B 空间，余弦相似度算出来的就是数学上恰好算出的那个东西。

所以每一次嵌入模型升级，都强制把新查询要对比的整个语料全量重嵌一遍。这就是重建索引税（the reindex tax），也是生产 RAG 里最被低估的一项成本。原始嵌入费用其实是最便宜的部分：1000 万 token 的语料用 `text-embedding-3-small`，按每百万 token 0.02 美元算大约 0.20 美元（2026 年年中价格，Batch API 另有 50% 折扣）。[^3] 10 亿 token 的企业语料是 20 美元。贵的是迁移窗口——那段时间你要么在提供陈旧结果，要么在并行跑两套索引。

真正管用的做法是**蓝绿迁移（blue-green migration）**，从 Web 运维借来的：在旧索引旁边把新索引建好，用一组固定的黄金查询（golden query）验证它，然后用别名原子地切换流量。

![一张横向流程图：左侧旧索引在服务线上流量，右侧新索引并行构建，中间是黄金查询校验器在对比两套索引，顶部一条珊瑚色的别名箭头在割接时刻从旧指向新](../../../ai-eng/part-6-retrieval-augmented-generation/images/08-blue-green-reindex.png)
*并行构建新索引；用黄金查询验证；切换一个别名完成割接。*

查询路径读一个功能开关（feature flag），路由到开关指向的那套索引。回滚是翻开关，不是重新部署代码：

```python
import os

EMBED_MODEL = os.getenv("EMBED_MODEL", "text-embedding-3-small")
EMBED_DIMENSIONS = int(os.getenv("EMBED_DIMENSIONS", "1536"))
USE_V2_EMBEDDINGS = os.getenv("USE_V2_EMBEDDINGS", "false").lower() == "true"

def get_query_index() -> str:
    return "index_v2" if USE_V2_EMBEDDINGS else "index_v1"
```

切换之前先跑一次黄金查询重叠检查：拿 50 到 200 条已知答案正确的查询，在两套索引上各跑一遍，量 top-10 结果集的重叠率。有一份公开记录的 `ada-002` 到 `text-embedding-3-large` 迁移，把 82% 的重叠率当作放行信号。[^4] 重叠率低不一定坏（更好的模型在边缘查询上本来就*应该*翻出不同的文档），但它是让你在翻开关之前手工看 diff 的信号。割接后两套索引都保持在线 24 到 48 小时，这样回滚不用重新嵌入。

还有一个更小的重建索引杠杆值得知道：`text-embedding-3` 系列支持 `dimensions` 参数，通过 Matryoshka 表示学习（Matryoshka representation learning）截断嵌入。传 `dimensions=256` 而不是默认的 1536，向量存储和 ANN 搜索成本能降 80% 以上，而在从业者基准里召回损失不到 1%。[^5] 这不是免费的画质，但对大多数 RAG 负载来说，这一个百分点的召回是值得花的地方。

什么会触发全量重建：换嵌入模型、换分块策略，或者两者都换。什么不会：给已有向量加一个新的元数据字段、更新文档内容（有哈希闸门，只有变动的分块会重嵌）。

## 多租户隔离：在检索前过滤，而不是检索后

最常见的多租户 RAG bug 在教程里长这样，在生产代码里也长这样：

```python
# 切勿这样做
results = vector_db.query(embedding=q, top_k=10)
results = [r for r in results if r.metadata["tenant_id"] == user.tenant_id]
```

这是安全表演（security theater）。等过滤器执行时，其他租户的文档早已被排序、打分，最坏情况下已经进了 LLM 的上下文。就算你把它们从响应里剔掉，模型也已经用它们调理过答案了。对抗性文档的情况更糟：针对后置过滤 RAG 系统的提示注入（prompt injection）攻击成功率约 80%，因为恶意指令在 LLM 层就已经执行了，而应用层的过滤器还没触发。[^6]

规则很简单：**在向量检索开始之前就把隔离做掉。** 有三种模式能做到，选哪种取决于租户数量和授权复杂度。

**按租户划分命名空间（结构化隔离）。** 给每个租户一个物理上独立的命名空间（namespace）、集合（collection）或分片（shard）。查询路径把 `tenant_id` 传进去，用来选*搜哪个*索引，永远不是作为共享索引内部的过滤条件。这样没有任何过滤表达式可以被配错而跨租户泄漏——因为压根没有可配错的东西。Pinecone 在 Enterprise 套餐下每个索引最多支持 100,000 个命名空间，Standard 是 25,000（2024 年 12 月发布）。[^7] Weaviate 用每租户独立分片实现这件事，Milvus 提供 database/collection/partition-key 三级层次，其中 database 层能给你完整的 RBAC 和物理资源组。[^8] 租户数在 1 万（10K）以下、且对数据硬隔离有要求时，默认选这个。

**PostgreSQL 行级安全（内核级强制）。** 如果你本来就在用 pgvector，行级安全（row-level security，RLS）把租户检查从应用代码下推到数据库内核：

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id'));
```

每次查询前先执行 `SET app.tenant_id = 'acme'`。将来某次代码改动把应用层过滤器删了，数据库仍然拒绝返回其他租户的行。据报告，在共享负载下 RLS 相比每租户一个独立数据库，能把隔离基础设施成本降最多 30%。[^9] 陷阱是连接池污染：如果一条池化连接归还时 `app.tenant_id` 还留着上一个请求的值，下一个请求就会静默地跑在错误租户的数据上。归还连接给池之前一定重置会话变量（`DISCARD ALL`），再加一道前置断言：`current_setting('app.tenant_id')` 必须与已认证的主体一致。

**用授权服务做前置 ACL 过滤。** 当租户数超过 10 万（100K），或者授权粒度在文档级而非租户级时，先取回允许的文档 ID，再约束向量搜索：

```python
allowed_doc_ids = auth_service.get_authorized_documents(user_id)
results = vector_db.query(
    embedding=query_embedding,
    where={"doc_id": {"$in": allowed_doc_ids}},
    limit=10,
)
```

`$in` 子句在 5 万个 ID 左右开始撑不住（多数向量数据库对超大 `$in` 表达式要么拒绝要么退化），所以要搭配一个真正的授权框架（Cerbos、Permit.io），把策略逻辑从应用代码里挪出去。

有一个漏洞横穿这三种模式：**过滤器注入（filter injection）**。如果你过滤表达式里的 `tenant_id` 值来自请求体，攻击者可以传 `{"$ne": "their-tenant"}`，把一个限定租户的过滤器改写成全租户查询。`tenant_id` 只能来自己认证的会话，绝不能来自用户输入。在服务端校验它，任何与已认证主体不符的值一律拒绝。

基于 agent 的检索还有一个更隐蔽的陷阱：**混淆代理（confused deputy）**。agent 代表用户去检索时，agent 自己的服务凭证通常比用户的权限大。没有限定范围的凭证，用户就会拿到他从无权查看的文档，而且没有任何授权检查会触发——因为对 agent 来说每次检查都通过了。请把用户范围的 token 贯穿整条检索链路，而不是让 agent 拿自己的身份去检索。

## 元数据过滤：HNSW 的召回悬崖

租户隔离用结构化手段解决之后，你还有一堆合理的元数据过滤：`doc_type = "policy"`、`language = "fr"`、`published_after = 2025-01-01`。这些看着无害。在基于图的 ANN 索引上，它们不是。

两种策略，没有一种显而易见：

- **后置过滤（post-filter）。** 先跑 ANN 搜索，再对结果套过滤器。如果过滤器只放行 5% 的文档，取 top 100 剩下来大约 5 条命中，远低于你 top-10 的目标。想拿到 10 条有用结果就得取几百条，延迟预算直接爆。
- **前置过滤（pre-filter）。** 先算出匹配集合，只在集合内搜索。过滤后集合小（约 1 万文档以下）时，暴力扫描是真的快，因为数组能塞进缓存。集合大时它线性扩展，慢到不可接受。

真正的问题是图断连（graph disconnection）。HNSW 是一张可导航小世界图（navigable small-world graph），你从入口点出发，沿着最近邻一路走。如果标准的前置过滤策略在遍历时跳过不匹配的节点，图就可能断连：从入口点到包含过滤命中结果的那片区域之间没有路径。当过滤条件与查询向量*负相关*时，这种退化最严重。设想查“钻石戒指”再加一个 50 美元以下的价格过滤。廉价商品在嵌入空间里离“钻石戒指”很远，于是图想走的那条路，永远不会靠近过滤器放行的那片区域。[^10]

Weaviate 1.27（2024 年 11 月）和 Elasticsearch 9.1 提供了 **ACORN**，这个算法用两跳扩展解决它：当遍历落到一个被过滤器拒绝的节点上，ACORN 直接看该节点的邻居，在不把失败节点算作候选的前提下保住连通性。在负相关过滤、选择率（selectivity）20% 的场景下，ACORN 相比旧策略在同等召回下吞吐量高最多 10 倍；ACORN 论文报告在固定召回下依选择率不同有 2 倍到 1000 倍的提升。[^10][^11] 现代向量数据库会判断 ACORN 何时有帮助，选择率高时自动切回标准 HNSW。

结论是：别把过滤当成“显然很便宜”就一笔带过。用真实语料测真实过滤选择率；如果你的数据库不支持 ACORN 式的过滤搜索，那凡是接近授权的场景，都优先用结构化隔离（独立命名空间）而不是高基数过滤表达式。

## 分阶段延迟预算

2 秒的 p95 SLA 不是一个数字，而是一份你要在四个串行阶段之间分配的预算。不按阶段做预算的工程师，只会在生产环境、负载之下的 p95 上，在最糟糕的那一天发现瓶颈。

| 阶段 | p50 目标 | p95 目标 | 说明 |
|---|---|---|---|
| 查询嵌入 | 30-50ms | 80-120ms | 缓存命中：3-15ms；本地模型：5-20ms |
| 向量检索 | 15-30ms | 50-150ms | 随语料规模和过滤复杂度增长 |
| 重排序（可选） | 50-150ms | 200-400ms | 存在时吃掉最多 65% 的检索预算 |
| LLM 生成 | 500-1000ms | 1500-2000ms | 主导 p95；用户感受到的是 TTFT |
| **端到端** | **~600ms** | **~2.5s** | 常见的企业 RAG 目标 |

各阶段区间数据的来源：针对 PyTorch 与 Anthropic 系 RAG 流水线的从业者架构分析。这些分析给出的典型向量数据库查询会额外带来 50-300ms 延迟，足以把语音流水线推过大约 200ms 的轮次间隔（turn gap）——超过这个间隔，听起来就不像自然对话了。[^12]

这张表里有两件事会让人意外。第一，生成吃掉 80% 的预算，所以检索加嵌入合计不能超过约 300ms，否则就开始切 SLA 了。第二，cross-encoder 重排序器听起来便宜，却悄悄消耗掉大部分非生成部分的余量。top-20 候选过一遍 BERT 级别的重排序器，p95 是 200-400ms。如果检索预算吸收不了，重排序器要么并行化，要么降级到 MiniLM-L6，要么在它不划算的查询上干脆不用。

各阶段的优化杠杆：

- **嵌入：** 把查询嵌入缓存在 Redis 里，键用归一化后的查询串，TTL 1 小时；命中率 >80% 能把该阶段 p95 从 80ms 降到 15ms。[^13] 或者用本地 sentence-transformers 模型，彻底省掉一次网络往返，拿 GPU 显存换延迟下限。
- **检索：** 调 HNSW 的 `ef` 参数（搜索扩展因子）；`ef` 越低越快，代价是召回。在黄金查询集上盯住召回，别为了调 SLA 把结果调成垃圾。
- **重排序：** 重排序前把候选集截到 top-20；关键词重的查询里 ANN 已经打得很准，就跳过重排序；延迟比边际召回更重要时，换更小的 cross-encoder。
- **生成：** 流式输出 token。首 token 500ms 对用户来说就是快，哪怕总生成跑到 2 秒。绝不要让流式路径阻塞在检索后的收尾工作上；引用标注和忠实度检查并行做。

两条架构规则压在分阶段预算之上。**入库永远不在查询路径上。** 文档通过队列（Redis Streams、Kafka）异步处理。查询路径只读已提交的索引，永远不阻塞在一次嵌入 API 调用上。违反这条规则，只要一波文档变更打进流水线，延迟尖峰就是必然的。

**给每个阶段设熔断器。** 500 QPS 时，一个慢下来的嵌入服务商、或者一个 GPU 压力下的重排序器，会让每个请求都退化。给每个阶段定硬超时和明确的兜底：嵌入超过 200ms，退到缓存或近似结果；重排序超过 300ms，直接返回 ANN 结果。一个阶段失败应该是优雅降级，而不是级联进 SLA。

要让这份预算真正可执行，还需要看板和分阶段 p50/p95 监控；架构尺度上的视图在 HLD 那边：[大规模向量搜索](../../hld/part-9-ai-ml-system-design/02-vector-search-at-scale.md)。本章讲的是预算这条原则，那边讲的是证明预算在真实流量下站得住的遥测数据。

## 参考文献

[^1]: Tian Pan, "Embedding Models in Production: Versioning and Index Drift," tianpan.co, April 2026. https://tianpan.co/blog/2026-04-09-embedding-models-production-versioning-index-drift

[^2]: Hou C., "Incremental Re-indexing and the Embedding Pipeline Nobody Talks About," Prompt/Deploy, April 2026. https://prompt-deploy.beehiiv.com/p/incremental-re-indexing-and-the-embedding-pipeline-nobody-talks-about

[^3]: OpenAI, "API Pricing: Embeddings," developers.openai.com, accessed June 2026. https://developers.openai.com/api/docs/pricing

[^4]: Hou C., "Incremental Re-indexing and the Embedding Pipeline Nobody Talks About," Prompt/Deploy, April 2026; the same walkthrough documents an `ada-002` to `text-embedding-3-large` migration using 82% golden-query overlap as the cutover signal. https://prompt-deploy.beehiiv.com/p/incremental-re-indexing-and-the-embedding-pipeline-nobody-talks-about

[^5]: Markaicode, "Reduce embedding cost with OpenAI's `dimensions` parameter," markaicode.com, 2026. https://markaicode.com/integrate/reduce-embedding-cost-openai/

[^6]: Tian Pan, "Vector Store Access Control: The Row-Level Security Problem Most RAG Teams Skip," tianpan.co, April 2026. https://tianpan.co/blog/2026-04-17-vector-store-access-control-rag-rls

[^7]: Pinecone, "Namespaces and multitenancy," Pinecone docs and December 2024 release notes. https://docs.pinecone.io/

[^8]: Robert Guo, "Designing Multi-Tenancy RAG with Milvus: Best Practices for Scalable Enterprise Knowledge Bases," Zilliz Blog, December 2024. https://zilliz.com/blog/build-multi-tenancy-rag-with-milvus-best-practices-part-one

[^9]: Markaicode, "PostgreSQL for RAG: cost and isolation patterns," markaicode.com, 2026. https://markaicode.com/usecases/postgresql-for-rag/

[^10]: Weaviate Engineering, "How we speed up filtered vector search with ACORN," Weaviate Blog, November 2024. https://weaviate.io/blog/speed-up-filtered-vector-search

[^11]: Patel et al., "ACORN: Performant and Predicate-Agnostic Search Over Vector Embeddings and Structured Data," arXiv:2403.04871, 2024. https://arxiv.org/abs/2403.04871

[^12]: Practitioner architecture analyses of PyTorch- and Anthropic-based RAG pipelines, markaicode.com, 2026, reporting 50-300ms typical vector-store query overhead in voice pipelines. Secondary sources; treat the ranges as directional. https://markaicode.com/

[^13]: Markaicode, "RAG architecture with Grafana: cache hit rate and observability," markaicode.com, 2026. https://markaicode.com/architecture/rag-architecture-with-grafana/
