---
title: "AI 基础设施模式"
description: "四类 AI 特有的基础设施组件：生产环境里的向量存储、语义缓存、agent 长时任务的持久队列，以及嵌入流水线。通用部分一律交叉链接到 HLD。"
slug: 07-ai-infrastructure-patterns
part: 10
chapter: 7
difficulty: intermediate
prerequisites:
  - part-6-retrieval-augmented-generation/02-embeddings-and-vector-search
  - part-6-retrieval-augmented-generation/08-production-rag
  - part-7-agent-engineering/08-long-running-agents
  - part-7-agent-engineering/06-agent-memory-and-task-state
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# AI 基础设施模式

你的 RAG 系统正在拿两周前的数据作答，而你毫无察觉。你的语义缓存（semantic cache）正兴高采烈地把用户 A 的账户余额返回给用户 B，只因为两人的问题余弦相似度是 1.0。你的 agent 把同一封邮件重发了五遍，因为 worker 跑到一半崩了，重试是从头开始的。你的 IVFFlat 索引召回率在过去一周的写入里悄悄从 0.98 滑到 0.85，监控里没有任何东西吭声。

本章讲的是四块基础设施——它们的故障模式带着 AI 特色，值得专门搞清楚。不是 Postgres，不是 Redis，不是 Kafka。那些都有成熟的通用指南，归 HLD Handbook 管。这里讲的是叠在上面的那层 AI 特有增量：向量存储（vector store）、语义缓存、agent 任务队列，以及嵌入流水线。

这四块共有的张力是新鲜度对成本。每个组件都在权衡同一件事：新数据多快能出现，以及为此要花多少算力或存储。共有的反模式是把它们当成通用兄弟组件的即插即用替代品；AI 特有的故障模式不会出现在通用监控里。

## 生产环境里的向量存储

向量存储持久化嵌入向量，对外暴露近似最近邻（approximate nearest neighbor，ANN）查询接口，并维护一个用召回率（recall）换查询速度的索引。它的运维生命周期与关系型数据库有三处不同，值得知道。

**索引类型及其运维画像。**

pgvector 的 HNSW（截至 v0.8.2，2026 年 2 月）支持插入而不必全量重建。运维上要紧的参数有三个：`m`（默认 16，每个节点的边数，调高召回更好但更吃内存）、`ef_construction`（默认 64，构建精度）、以及 `hnsw.ef_search`（默认 40，查询时的扫描宽度，按连接设置）。[^1]

pgvector 的 IVFFlat 对插入算友好（online-friendly），但它的聚类中心只在建索引时算一次。随着写入累积、数据分布漂移，聚类中心变得不再有代表性，召回率就往下掉。从业者的报告说：每周新增 10 万个嵌入向量时，IVFFlat 的召回率会在一周内从 0.98 掉到 0.85，只有跑完 `REINDEX INDEX CONCURRENTLY` 才恢复。每新增 10 万行、或者数据集翻倍，就安排一次重建。[^1]

Pinecone 的 serverless 架构完全绕开了面向用户的索引管理。写入先落到 S3 上的 WAL，100ms 内返回确认，再经内存里的 memtable 在几秒内变得可查。压实（compaction，L0 → L1 → L2 各层 slab）是异步且透明的。算法选择（小 slab 用 Ananas、中等用 PQFS、大的用 IVF）按每个 slab 自动决定，不用你调。[^2]

**2,000 维的上限。** pgvector 的 HNSW 索引在 `vector` 类型上最多 2,000 维，在 `halfvec` 上最多 4,000 维。OpenAI 的 text-embedding-3-large 最多输出 3,072 维。用它就得三选一：通过 `dimensions` API 参数截断，存成 `halfvec(3072)` 并在查询时转型，或者干脆放弃 HNSW 索引、退回顺序扫描（对全表 O(n)）。[^1] 这是有文档记录的硬约束，不是配置疏漏。

**维度变更的迁移。** 换嵌入模型意味着整个向量列都得重建。`vector(1536)` 和 `vector(1024)` 是不兼容的类型。标准做法是双列：加一个目标维度的新列，增量灌数据，在旧列继续服务流量的同时给新列建 HNSW 索引，然后在应用层切换、再删掉旧列。把嵌入模型升级当成破坏性的 schema 迁移来对待。

决策规则：已经在跑 PostgreSQL、向量数不到约 500 万、QPS 中等的团队，用 pgvector。出现下面任一情况就该升级到专用向量数据库（Pinecone serverless、Qdrant）：向量超过 1,000 万，查询 SLA 要求 p95 低于 20ms，写入吞吐超过每秒几千个向量，或者你需要每租户独立的索引、且不允许跨租户泄漏。Timescale 2025 年 5 月在 5,000 万个 768 维向量上的基准显示，pgvectorscale 在 99% 召回率下达到 471 QPS，而 Qdrant 是 41 QPS；但 pgvectorscale 建索引花了 11 小时，Qdrant 是 3.3 小时。[^3]

HNSW、IVFFlat 的底层机制，以及嵌入与索引的选型，见[向量嵌入与向量检索](../part-6-retrieval-augmented-generation/02-embeddings-and-vector-search.md)。本章只讲生产环境里的运维。

## 语义缓存

语义缓存与精确匹配的键值缓存有一个根本区别：缓存键不是一个逐字节相同的字符串，而是对已存提示词嵌入做一次 ANN 查询得到的结果。查找流程是：

1. 把进来的查询做嵌入
2. 在缓存的向量索引里找最近的已存提示词嵌入
3. 把距离与阈值比较
4. 命中：返回存好的 LLM 响应，不调用模型。未命中：调用 LLM，然后存下新的提示词嵌入、响应和元数据

GPT Semantic Cache 论文（arXiv 2411.05276，2024 年 12 月）给出了具体延迟：阈值 0.8 下的缓存命中约 13ms 返回，而 LLM API 调用约 866ms。各类 FAQ 场景的缓存命中率在 61.6%–68.8% 之间，命中的正确率为 92.5%–97.3%。[^4] Redis 的文档声称在 FAQ 负载上能省下 30% 以上的 token 开销，且质量没有可测量的回退。[^5]

> [!WARNING]
> **阈值问题是这里的核心工程挑战。** UC Berkeley 的 vCache 论文（arXiv 2502.03771，2025 年 2 月）证明，正确命中与错误命中的相似度分布高度重叠：在 SemCacheClassification 基准上，两者均值分别是 0.84 和 0.85。[^6] 没有任何一个静态阈值能在所有提示词上把它们干净地分开。阈值放太松，你会输出错误答案；收太紧，命中率直接塌掉。

生产工具（GPTCache、RedisVL、LiteLLM）里的从业者默认做法是一个静态全局阈值：窄领域通常用 0.8 余弦，通用问答用 0.9–0.95。有文档记录的失效模式是：查询分布越多样，错误率就单调上升，找不到稳定的静态阈值。vCache 的做法是按每个嵌入学习动态阈值，从观测到的（相似度，正确性）对里学，在错误率相当的前提下命中率最高提升到 2 倍，在 SemCacheLMArena 基准上错误率降到原来的 1/6。[^6]

一个可用的 RedisVL 实现：

```python
from redisvl.extensions.cache.llm import SemanticCache
from redisvl.utils.vectorize import HFTextVectorizer

# distance_threshold 用 Redis 的 COSINE 单位 [0-2]；0.1 = 严格，0.5 = 宽松
llmcache = SemanticCache(
    name="llmcache",
    redis_url="redis://localhost:6379",
    distance_threshold=0.1,
    vectorizer=HFTextVectorizer("redis/langcache-embed-v2"),
)

def ask_with_cache(question: str, llm_fn) -> str:
    results = llmcache.check(prompt=question)
    if results:
        return results[0]["response"]
    answer = llm_fn(question)
    llmcache.store(prompt=question, response=answer)
    return answer
```

另一个坑每个团队都会踩：跨租户缓存污染。两条嵌入相似度很高、但正确答案因人而异的提示词被混为一谈。用户 A 的“我的账户余额是多少？”和用户 B 的同一句问法余弦相似度 1.0，缓存就把 A 的答案给了 B。修法没有商量余地：用元数据过滤收窄缓存条目的作用域，把 tenant_id、user_id、locale、模型版本都做成 TAG 字段。Redis Search 能在一次查询里把 ANN 查找和 TAG 过滤结合起来。[^5]

查询分布极其多样、问题很少重复（企业文档问答、多轮 agent 状态），或者响应必须反映实时数据，或者响应依赖无法做成可过滤元数据的用户特有状态——这些情况下不要用语义缓存。

## 长时运行的 agent 任务队列

一个典型的 web 请求一秒内就结束。一个 agent 任务（多步推理、工具调用、人工确认、带重试的 LLM 推理）可能跑几分钟到几小时。标准任务队列（job queue，如 Celery + Redis、BullMQ + Redis）提供任务分发和重试，但不保存持久的工作流状态：worker 执行到一半崩了，任务就从头再来。对一个已经调了三次 LLM、执行了两次工具调用、正等着人工审批的 agent 来说，重开既烧钱，还可能把有副作用的工具重复执行一遍。

AI 特有的要求是**持久执行（durable execution）**：任何失败（崩溃、发布、网络分区）之后都能精确重建 agent 当时的位置，且不重跑已完成的步骤。通用队列机制在 HLD Handbook 里讲；对 AI 来说变的是检查点（checkpoint）的粒度。

**Temporal** 是生产级的答案。[^7] 核心原语是 Workflow（用 `@workflow.defn` 装饰的函数），它以长期存活的状态机形式运行。非确定性操作（LLM 调用、工具调用、读环境）统统包成 Activity。每个 Activity 的结果都追加进一份不可变的 Event History。worker 崩了，接替的 worker 会重放 Event History 重建工作流状态，只重跑未完成的 Activity，已完成的直接跳过。

AI agent 的写法是这样的：

- LLM 调用和工具调用都跑成 Activity（失败自动重试）。
- 对话历史、当前工具选择、确认状态放在 Workflow 的实例变量里（跨重启持久）。
- 用户输入通过 Signal 进来（发给运行中 Workflow 的异步消息）。
- 对话状态通过 Query 暴露（对运行中 Workflow 的同步读取）。

```python
from temporalio import workflow, activity
from dataclasses import dataclass
from datetime import timedelta

@dataclass
class AgentInput:
    goal: str

@activity.defn
async def call_llm(prompt: str) -> str:
    # 非确定性调用包成 Activity。结果持久化到 Event History；
    # 崩溃恢复时读取该结果，而不是重新调用模型。
    raise NotImplementedError  # 替换成真正的 LLM 调用

@workflow.defn
class AgentWorkflow:
    def __init__(self) -> None:
        self._inbox: list[str] = []

    @workflow.run
    async def run(self, inp: AgentInput) -> str:
        await workflow.wait_condition(lambda: bool(self._inbox))
        user_msg = self._inbox.pop(0)
        return await workflow.execute_activity(
            call_llm, user_msg,
            start_to_close_timeout=timedelta(seconds=30),
        )

    @workflow.signal
    async def user_message(self, msg: str) -> None:
        self._inbox.append(msg)
```

两个值得知道的运维细节。**Continue-As-New** 对长时运行的 agent 是必须的，因为 Event History 会无界增长：消息数超过阈值时，用一个 LLM Activity 生成两句摘要，把它作为新一次 Workflow Execution 的起始状态，然后关掉旧的。**任务队列隔离**能防止一批廉价的嵌入任务把昂贵的长时 agent Workflow 饿死：按 agent 类型分队列，用 Kubernetes HPA 按队列深度自动扩缩容（autoscaling）；LLM worker 是 I/O 密集型，不是 CPU 密集型。

对更简单、无状态的任务（发完就不管的嵌入作业、顺序链式调用），Celery + Redis 就够了。要避免的运维失误是：得配上 `task_acks_late=True`、`task_reject_on_worker_lost=True` 和一个死信队列（dead-letter queue）。少了这些，Celery worker 在任务中途崩溃会静默重启该任务，可能把有副作用的工具再执行一遍。队列本身的机制（broker 配置、重试退避、DLQ 模式），见 HLD 的[消息队列与流式处理](../../hld/part-2-building-blocks/08-message-queues-streaming.md)一章。

## 嵌入流水线

嵌入流水线把源文档转成向量表示，并 upsert 进向量存储。InfoWorld 的说法非常准：“嵌入流水线本质上是数据工程问题，不是一门全新的 AI 学科。它的核心仍然是 ETL，只不过目的地从数据仓库换成了嵌入和向量存储。”[^8]

三个阶段，对应到 ETL：摄取（拉原始文档、检测变化）、分块（切分、归一化）、建索引（对每个块做嵌入、upsert 进向量存储）。

**批处理与增量。** 批处理按计划把整个语料重嵌一遍。实现简单；新鲜度滞后等于批处理间隔。按 OpenAI text-embedding-3-small 的价格（每百万 token 0.02 美元，2026 年 6 月）、每块约 500 个 token 算，嵌 100 万份文档每轮约 10 美元；1,000 万份是每轮 100 美元。源文档很少变动时，这可以接受。

增量则用变更数据捕获（Change Data Capture，CDC）机制找出上次运行以来变过的文档，只重嵌这些。这需要一份文档清单（manifest：内容哈希 + 时间戳 + 嵌入模型版本）。报告的数字是：陈旧时间 2–5 秒，相比全量重建索引成本降低 60%。[^9]

```python
import hashlib
from dataclasses import dataclass
from typing import Optional

EMBEDDING_MODEL_VERSION = "text-embedding-3-small-v1"

@dataclass
class ManifestEntry:
    doc_id: str
    content_hash: str
    model_version: str
    last_embedded_at: float

def compute_hash(content: str) -> str:
    return hashlib.sha256(content.encode()).hexdigest()

def needs_re_embedding(doc_id: str, content: str, manifest: dict) -> bool:
    """True if document is new, changed, or was embedded with a different model."""
    entry: Optional[ManifestEntry] = manifest.get(doc_id)
    if entry is None:
        return True
    if entry.content_hash != compute_hash(content):
        return True
    if entry.model_version != EMBEDDING_MODEL_VERSION:
        return True
    return False
```

**模型版本是个不出声的正确性问题。** 向量存储里的每个块都必须打上产出它的嵌入模型名与版本。升级模型之后，旧模型的向量与新模型的向量在几何上不可比。一个混着两代向量的索引会带来无声的召回退化：新模型的查询向量不会聚到旧模型文档向量附近。InfoWorld 那篇文章说：“不同版本产出的向量之间不存在可靠的比较关系。你不能把它们当作可互换的向量，安全地跨版本检索。”[^8]

迁移纪律和向量存储的维度变更一样：加一个新向量列，异步用新模型跑一遍全部文档，在新列上建索引，切换查询层，删掉旧列。迁移窗口期内，旧列继续服务线上流量。

生产环境嵌入流水线的组件，全部通过队列解耦：

- **源连接器**，盯着文档存储（S3、Confluence、Postgres、Git）的变化
- **清单存储**，按文档记录哈希 + 模型版本 + 最后嵌入时间
- **分块服务**，分块配置带版本；配置一改就触发重新分块
- **嵌入服务**，批量调用（OpenAI 单次请求最多支持 2,048 条输入）
- **向量存储写入器**，做批量 upsert（首次加载用 `COPY`，增量用 `INSERT ON CONFLICT DO UPDATE`）
- **阶段之间的队列**，这样嵌入服务临时不可用时不会丢文档

造成生产事故最多的架构决策：把嵌入流水线和在线查询路径耦合在一起。批量摄取卡住了，检索也该照常可用。流水线的健康指标要和查询路径指标分开监控：嵌入吞吐（块/分钟）、上次成功运行的时间戳、待处理队列深度。最新文档的 `last_embedded_at` 超过预期流水线间隔的 2 倍还没往前走，就报警。

## 这四块共同说明了什么

这四块有个共同点：它们的通用兄弟单独用时都没问题，但那些 AI 特有的故障模式（嵌入版本悄悄混用、缓存误命中、agent 跑一半丢状态、IVFFlat 召回漂移）在标准监控里全是隐形的。语义缓存跨了租户、IVFFlat 索引退化了、嵌入流水线卡住了——你不会收到 HTTP 500。你收到的是 HTTP 200，外加一份微妙地错的内容，这和 [SLO 与事故响应](06-slos-and-incident-response.md)里每一种故障模式是同一个套路。

上线前就把 AI 特有的监控建起来：向量召回抽检、缓存命中正确性审计（抽样缓存响应，和现场重调 LLM 的结果对比）、嵌入流水线新鲜度告警、持久工作流状态巡检。通用监控栈这些一个都抓不到。

基础通用设施（队列机制、Redis 缓存、Postgres 扩展、Kafka、Kubernetes），参考 [HLD Handbook](../../hld/)。本章刻意写得窄：只讲 AI 特有的那部分增量。这部分增量为数不多，却承重，2025–2026 年大多数生产事故就出在这一小块重叠区域里。

## 参考文献

[^1]: pgvector 贡献者，"pgvector CHANGELOG and README"，版本 0.5.0 至 0.8.2，2023 年 8 月至 2026 年 2 月，https://github.com/pgvector/pgvector/blob/master/CHANGELOG.md
[^2]: Pinecone，"How Pinecone Works: Architecture and Engineering Deep Dive"，2025–2026 年，https://www.pinecone.io/how-pinecone-works/
[^3]: Timescale，"pgvector vs Qdrant: 50M Vector Benchmark"，2025 年 5 月，https://www.timescale.com/blog/pgvector-vs-qdrant
[^4]: Sajal Regmi 与 Chetan Phakami Pun，"GPT Semantic Cache: Reducing LLM Costs via Semantic Embedding Caching"，arXiv:2411.05276，2024 年 12 月，https://arxiv.org/html/2411.05276v3
[^5]: Redis，"Redis semantic cache"，官方文档，2025–2026 年，https://redis.io/docs/latest/develop/use-cases/semantic-cache/
[^6]: Luis Gaspar Schroeder 等，"vCache: Verified Semantic Prompt Caching"，arXiv:2502.03771，2025 年 2 月，https://arxiv.org/html/2502.03771v3
[^7]: Temporal，"How To Build a Durable AI Agent with Temporal and Python"，官方教程，2025 年 7 月，https://learn.temporal.io/tutorials/ai/durable-ai-agent/
[^8]: Chisom Nwokwu，"Embedding pipelines are the new ETL"，InfoWorld，2026 年 6 月，https://www.infoworld.com/article/4181232/embedding-pipelines-are-the-new-etl.html
[^9]: markaicode.com，"Vector Database Architecture with LlamaIndex: Production Blueprint for High-Throughput RAG"，2026 年，https://markaicode.com/architecture/vector-database-architecture-with-llamaindex/
