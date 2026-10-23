<div align="center">

<img src="logo.svg" alt="Handbook Academy" width="140" />

# The Engineering Handbook

**给做软件的人读的免费开源工程书。**<br/>
系统设计、数据结构与算法、AI 工程，每一本都从头讲到尾，永久免费阅读。

[![License: CC BY-SA 4.0](https://img.shields.io/badge/license-CC%20BY--SA%204.0-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![CI](https://github.com/handbook-academy/engineering-handbook/actions/workflows/content-ci.yml/badge.svg)](https://github.com/handbook-academy/engineering-handbook/actions)
[![GitHub stars](https://img.shields.io/github/stars/handbook-academy/engineering-handbook?style=social)](https://github.com/handbook-academy/engineering-handbook/stargazers)

**[在 handbook.academy 在线阅读](https://handbook.academy)**

[HLD Handbook](#the-hld-handbook) · [DSA Handbook](#the-dsa-handbook) · [AI Engineering Handbook](#the-ai-engineering-handbook) · [从哪里开始](#从哪里开始) · [学习计划](#学习计划) · [参与贡献](#参与贡献) · [常见问题](#常见问题)

</div>

---

## 目录

- [关于本项目](#关于本项目)
- [三本手册](#三本手册)
- [在线阅读](#在线阅读)
- [适合谁读](#适合谁读)
- [从哪里开始](#从哪里开始)
- [The HLD Handbook](#the-hld-handbook)
  - [HLD 各部分速览](#hld-各部分速览)
  - [权衡库](#权衡库)
- [The DSA Handbook](#the-dsa-handbook)
  - [DSA 各部分速览](#dsa-各部分速览)
  - [模式决策页](#模式决策页)
  - [长篇题解](#长篇题解)
  - [交互组件规格](#交互组件规格)
- [The AI Engineering Handbook](#the-ai-engineering-handbook)
  - [AI Engineering 各部分速览](#ai-engineering-各部分速览)
- [学习计划](#学习计划)
- [书是怎么写出来的](#书是怎么写出来的)
- [参与贡献](#参与贡献)
- [仓库结构](#仓库结构)
- [开发环境](#开发环境)
- [常见问题](#常见问题)
- [许可证](#许可证)
- [引用](#引用)
- [致谢](#致谢)
- [社区](#社区)

---

## 关于本项目

The Engineering Handbook 是一套写给软件工程师的免费教材。每一本都是一门完整的课：每个主题都在页面上讲完，从第一性原理一直讲到团队在生产里踩过的坑。你在这里找不到大纲、链接清单或者“请去别处阅读”。

这些书以纯 Markdown 形式放在这个仓库里。它们在 GitHub 上渲染时，图、插图和脚注一个不少；同一份文本也撑起了 [handbook.academy](https://handbook.academy) 上的网站。全部内容以 [CC BY-SA 4.0](LICENSE) 授权，所以这些文字始终可以免费阅读、分享和改编。

这套书不一样的地方：

- **讲透。** 每一章都把概念讲清楚，用图和实例演算展示出来，比较备选方案，最后告诉你上生产要盯什么。
- **有观点。** 章节会给出推荐做法并说明理由。当行内人意见不一致时，我们把权衡摊开写，让你自己判断。
- **有出处。** 每个论断都有脚注指向一手资料：论文、RFC、官方文档和工程复盘。
- **跟得上当下。** 书里覆盖今天的技术栈：LLM 服务、RAG、agent、MCP、CRDT、边缘计算、FinOps 和后量子密码学。每一章都记录了最后更新时间。
- **开放。** 谁都可以提修改，每一处改动都要过 review。

## 三本手册

| 手册 | 讲什么 | 适合谁 | 入口 |
|---|---|---|---|
| **[HLD 手册](#the-hld-handbook)** | 如何设计大型软件系统：网络与存储、分布式系统理论、架构、可靠性、安全、端到端案例研究，以及 AI 系统设计。 | 面向系统设计面试，以及负责设计或运维后端系统的工程师。 | [章节](#hld-各部分速览) · [网站](https://hld.handbook.academy) |
| **[DSA 手册](#the-dsa-handbook)** | 按模式组织的数据结构与算法，LeetCode 题目用 Python 解答，多数另附 Java、C++ 与 Go 版本。 | 面向编程面试、竞赛，以及想把解题手感练利落的人。 | [章节](#dsa-各部分速览) · [网站](https://dsa.handbook.academy) |
| **[AI Engineering 手册](#the-ai-engineering-handbook)** | 如何在模型 API 之上构建并运营生产级 AI 产品：提示词、评测、上下文、检索、智能体、可观测性、安全、成本与产品。 | 面向要做 AI 功能的软件工程师。不需要训练模型。 | [章节](#ai-engineering-各部分速览) · [网站](https://ai-eng.handbook.academy) |

## 在线阅读

每本书都有自己的网站。每一章在那里都免费开放给所有人，不用登录。

| 站点 | 地址 | 有什么 |
|---|---|---|
| **首页** | [handbook.academy](https://handbook.academy) | 三本书的入口 |
| **HLD 手册** | [hld.handbook.academy](https://hld.handbook.academy) | HLD 手册 |
| **DSA 手册** | [dsa.handbook.academy](https://dsa.handbook.academy) | DSA 手册 |
| **AI Engineering 手册** | [ai-eng.handbook.academy](https://ai-eng.handbook.academy) | AI Engineering 手册 |

更习惯 GitHub？这份 README 里的每个章节链接都指向源文件，Mermaid 图直接在浏览器里渲染。

## 适合谁读

- **准备面试的工程师。** 编码轮靠 DSA，系统设计轮靠 HLD，面试本身那一套框架两本书里都有专门的部分。
- **想再上一个层级的工程师。** 把 Senior 和 Staff 的设计拉开差距的，正是 HLD 里分布式系统、可靠性和安全这几部分。
- **在做 AI 功能的工程师。** AI Engineering Handbook 讲的是怎么让一个 LLM 功能可靠、可度量、安全、成本可控。
- **自学出身的工程师。** HLD 的 Prerequisites 和 DSA 的 Foundations，覆盖了科班本该给你的那部分计算机科学。
- **转方向的工程师**，从前端转后端、后端转基础设施、软件转 AI，需要尽快建立系统层面的直觉。
- **老师和课程设计者**，想要一套完整课程当底子。许可协议允许。

**不适合的情况：** 如果你从没写过代码，先从 [CS50](https://cs50.harvard.edu/) 或 [The Odin Project](https://www.theodinproject.com/) 开始，之后再回来。这几本书讲的是跨语言通用的概念，而不是某门语言的语法。

## 从哪里开始

| 我想…… | 从这里开始 |
|---|---|
| 想快速尝尝系统设计是什么味道 | [可扩展性：让系统在增长中不崩](content/zh/hld/part-1-core-fundamentals/00-scalability.md) → [量级估算](content/zh/hld/part-1-core-fundamentals/04-back-of-envelope-estimation.md) → [设计一个 URL 短链接服务（TinyURL / bit.ly）](content/zh/hld/part-8-case-studies/00-url-shortener.md) |
| 准备系统设计面试 | [如何拆解一道系统设计题](content/zh/hld/part-1-core-fundamentals/05-how-to-approach-design-questions.md)，再看[面试准备计划](#interview-prep-sde1-to-sde2) |
| 想重新夯实分布式系统的基础 | [HLD 第 3 部分 —— 分布式系统理论](#hld-part-3) |
| 想更稳地过编程面试 | [数组：静态、动态与多维](content/zh/dsa/part-1-linear-data-structures/00-arrays.md) → [双指针：两端相向](content/zh/dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md) → [滑动窗口：变长窗口](content/zh/dsa/part-3-pointers-window-prefix/03-sliding-window-variable.md) |
| 想在两种方案之间做选择 | 设计问题看[权衡库](#权衡库)，算法问题看[模式决策页](#模式决策页) |
| 想做出一个站得住脚的 AI 功能 | [AI 工程师是什么（以及不是什么）](content/zh/ai-eng/part-0-start-here/00-what-an-ai-engineer-is.md) → [升级阶梯](content/zh/ai-eng/part-0-start-here/03-the-escalation-ladder.md) → [为什么没有评测就上不了线](content/zh/ai-eng/part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md) |
| 想设计规模化的 AI 系统 | [HLD 第 9 部分 —— AI 与 ML 系统设计](#hld-part-9)，以及 [AI 系统案例](#ai-系统) |
| 想按一个有截止日期的计划推进 | [学习计划](#学习计划)，从一个周末到完整课程 |

---

## The HLD Handbook

**如何设计大型软件系统：从网络一路讲到 LLM 服务。**

[网站](https://hld.handbook.academy) · [`content/hld/`](content/hld/) · [风格指南](STYLE_GUIDE.md)

HLD Handbook 先讲前置知识（网络、操作系统、数据库、API 设计），再顺着基础组件、分布式系统理论、数据系统和架构模式往上搭。之后讲真正把系统跑起来需要的东西：可靠性、运维和安全。案例研究用端到端设计把前面这些串起来，最后几部分讲 AI 系统设计，以及面试怎么应对。

**一章是怎么写出来的。** 每个教学章节都走同一条路：引言、从第一性原理讲概念、配图、一个完整实例、权衡、生产上的坑、参考文献。案例研究则换成面试的形状：需求、估算、API、数据模型、架构、深入探讨、扩展。

### HLD 各部分速览

点任意一个部分编号，直接跳到它的章节；也可以往下翻，逐个打开。

| 部分 | 内容简介 | 难度 |
|---|---|---|
| [0](#hld-part-0) | **前置基础** —— 网络、操作系统、数据结构、数据库、API 设计 | 入门 |
| [1](#hld-part-1) | **核心基础** —— 可扩展性、延迟、可用性、一致性、量级估算、权衡思维 | 入门到中级 |
| [2](#hld-part-2) | **构建模块** —— 负载均衡、缓存、队列、CDN、数据库、分片、发布订阅、限流、对象存储、地理索引、边缘计算 | 中级到进阶 |
| [3](#hld-part-3) | **分布式系统理论** —— 共识、一致性、时钟、CRDT、分布式事务 | 中级到进阶 |
| [4](#hld-part-4) | **数据系统** —— 存储引擎、OLTP/OLAP、数仓与数据湖、流处理、搜索、时序、图、向量、键值 | 中级到进阶 |
| [5](#hld-part-5) | **架构模式** —— 单体与微服务、事件驱动、CQRS、事件溯源、Serverless、BFF、绞杀者、六边形、多地域、多租户、CRDT 应用 | 中级到进阶 |
| [6](#hld-part-6) | **可靠性与运维** —— 可观测性、SLO、韧性、扩容、发布、混沌工程、事故响应、FinOps、平台工程 | 中级到进阶 |
| [7](#hld-part-7) | **规模化安全** —— 认证与授权、OAuth/OIDC、JWT、mTLS、密钥管理、DDoS/WAF、合规、供应链、隐私、后量子密码 | 中级到进阶 |
| [8](#hld-part-8) | **案例研究** —— 端到端设计，按主题分组；面试准备的重头戏 | 中级到进阶 |
| [9](#hld-part-9) | **AI 与 ML 系统设计** —— LLM 服务、RAG、向量检索、智能体、LLMOps、安全、推荐系统、多模态 | 中级到进阶 |
| [10](#hld-part-10) | **新兴模式** —— 绿色与可持续计算 | 中级 |
| [11](#hld-part-11) | **面试框架** —— RESHADED/PEDALS/ADEPT、需求、画图、权衡、公司风格、RFC | 中级到进阶 |

<details>
<summary><a id="hld-part-0"></a><strong>第 0 部分 · 前置基础</strong></summary>

**级别：** 入门 &nbsp;·&nbsp; **适合：** 没有 CS 学位、或想先确认自己的基础再继续往下走的工程师。

本手册其余部分默认你已经掌握的基础主题。如果你能讲清 TCP 的三次握手、进程与线程的区别、B 树的内部结构，以及为什么对容易重试的接口来说幂等 PUT 比非幂等 POST 更合适——这一部分可以直接跳过。

| # | 章节 | 内容简介 |
|---|---|---|
| 0.0 | [系统设计中的网络基础](content/zh/hld/part-0-prerequisites/00-networking-fundamentals.md) | 从实践角度走一遍 OSI 分层、TCP 与 UDP、HTTP/1.1 到 HTTP/3、DNS、TLS 1.3，以及 WebSocket 和 SSE 这类实时传输方式。 |
| 0.1 | [系统设计所需的操作系统基础](content/zh/hld/part-0-prerequisites/01-os-essentials.md) | 进程、线程、带真实延迟数字的内存层级、I/O 模型（epoll、kqueue、io_uring）以及文件系统内部机制——设计系统前必须掌握的基础。 |
| 0.2 | [分布式系统常用的数据结构](content/zh/hld/part-0-prerequisites/02-data-structures-for-systems.md) | 支撑现代基础设施的少数几种数据结构：哈希表、B 树、LSM 树、布隆过滤器、跳表和一致性哈希环。 |
| 0.3 | [系统设计中的数据库基础](content/zh/hld/part-0-prerequisites/03-database-fundamentals.md) | SQL、ACID、索引、查询执行，以及范式化与反范式化之争——讲到设计真实系统所需的深度。 |
| 0.4 | [API 设计基础：REST、GraphQL、gRPC 与那些难点](content/zh/hld/part-0-prerequisites/04-api-design-basics.md) | 资源建模、GraphQL 的 N+1 问题、gRPC 流式、版本化策略、幂等键，以及正确实现游标分页。 |

</details>

<details>
<summary><a id="hld-part-1"></a><strong>第 1 部分 · 核心基础</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 所有人——即使你已经熟悉这些主题也要读，因为后面全书都在用这里的词汇。

后面每一章都默认你已经掌握的词汇和推理习惯。“可扩展性”“一致性”“权衡”“量级估算”都在这里被严格定义，这样我们在后面使用它们时才有确切的含义。

| # | 章节 | 内容简介 |
|---|---|---|
| 1.0 | [可扩展性：让系统在增长中不崩](content/zh/hld/part-1-core-fundamentals/00-scalability.md) | 纵向扩展与横向扩展、无状态服务、读扩展与写扩展，以及扩展反而是错答案的时候。系统设计对话里绕不开的那套词汇。 |
| 1.1 | [延迟与吞吐量：真正重要的两个数字](content/zh/hld/part-1-core-fundamentals/01-latency-and-throughput.md) | 延迟与吞吐量的对比、Jeff Dean 的延迟数字表、长尾延迟（p50/p95/p99）、利特尔定律，以及如何定位任何系统中真正的瓶颈。 |
| 1.2 | [可用性与可靠性：几个九、SLO 与持续在线](content/zh/hld/part-1-core-fundamentals/02-availability-and-reliability.md) | 可用性数学、MTBF/MTTR、冗余模式、SLI/SLO/SLA、错误预算、依赖链数学，以及真实故障带来的教训。 |
| 1.3 | [一致性模型：读者到底看到什么](content/zh/hld/part-1-core-fundamentals/03-consistency-models.md) | 强一致性、最终一致性与因果一致性。读己之写、单调读，以及让一致性变得可处理的客户端视角与数据视角之分。 |
| 1.4 | [量级估算](content/zh/hld/part-1-core-fundamentals/04-back-of-envelope-estimation.md) | 2 的幂与 10 的幂、存储与带宽模板、QPS 算术，以及 Twitter 和 YouTube 规模的实际算例。这项面试技能在生产环境里同样重要。 |
| 1.5 | [如何拆解一道系统设计题](content/zh/hld/part-1-core-fundamentals/05-how-to-approach-design-questions.md) | 一套可复用的六步框架，应对 45 分钟的系统设计面试：厘清需求、量级估算、API 设计、高层设计、深入细节、权衡收尾。附逐分钟计划与各公司风格差异。 |
| 1.6 | [权衡思维](content/zh/hld/part-1-core-fundamentals/06-trade-off-thinking.md) | 每个设计决策都是权衡。如何把选择讲清楚、组织成结构、并为它辩护，以及如何识别不出手才是正确选择的时候。 |

</details>

<details>
<summary><a id="hld-part-2"></a><strong>第 2 部分 · 构件</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 任何要动手组装后端系统的人。这一部分就是积木清单。

对你搭建真实系统时会用到的零件做深入拆解：负载均衡器、缓存、队列、CDN、数据库（SQL 与 NoSQL）、分区、复制、发布/订阅、限流器、服务发现、blob 存储、地理空间索引和边缘计算。

| # | 章节 | 内容简介 |
|---|---|---|
| 2.0 | [负载均衡器：分散流量，吸收故障](content/zh/hld/part-2-building-blocks/00-load-balancers.md) | L4 与 L7 负载均衡、调度算法、健康检查，以及 Envoy、HAProxy、NGINX 和 AWS ALB/NLB 在生产中到底怎么工作。 |
| 2.1 | [反向代理与 API 网关：智能边缘](content/zh/hld/part-2-building-blocks/01-reverse-proxies-api-gateways.md) | TLS 终结、路由、认证、限流，以及为什么 API 网关不只是更聪明的负载均衡器。 |
| 2.2 | [内容分发网络：把字节搬到离用户更近的地方](content/zh/hld/part-2-building-blocks/02-cdns.md) | CDN 如何工作，边缘缓存、缓存键与失效，以及 Cloudflare、Akamai、Fastly 在实践中的差别。 |
| 2.3 | [缓存：从浏览器到数据库](content/zh/hld/part-2-building-blocks/03-caching.md) | 缓存层级、五种写入模式、淘汰策略（LRU/TinyLFU）、雪崩预防，以及为什么缓存失效是分布式系统里最难的问题。 |
| 2.4 | [SQL 数据库：无聊但胜出的技术](content/zh/hld/part-2-building-blocks/04-sql-databases.md) | B 树、MVCC、隔离级别、索引，以及为什么 PostgreSQL 和 MySQL 仍是大多数系统的正确答案。 |
| 2.5 | [NoSQL 数据库：选对非关系型工具](content/zh/hld/part-2-building-blocks/05-nosql-databases.md) | 键值、文档、宽列与图存储，以及 DynamoDB、Cassandra、MongoDB、Neo4j 在实践中的差异。 |
| 2.6 | [数据库分区与分片：当一个节点不够用时](content/zh/hld/part-2-building-blocks/06-database-partitioning-sharding.md) | 范围分区、哈希分区与一致性哈希分区、热点、重新分片，以及 Notion、Figma、Discord 在生产中如何切分数据。 |
| 2.7 | [数据库复制：让副本保持同步](content/zh/hld/part-2-building-blocks/07-database-replication.md) | 主从、多主与无主复制，同步与异步，故障切换，以及 Postgres、MySQL、Cassandra 和 DynamoDB 之间的差异。 |
| 2.8 | [消息队列与流式处理：大规模解耦](content/zh/hld/part-2-building-blocks/08-message-queues-streaming.md) | 队列与日志的区别，Kafka、RabbitMQ、SQS 的权衡，投递语义、分区、消费组，以及什么时候流式处理胜过请求-响应。 |
| 2.9 | [发布/订阅：扇出与事件驱动系统](content/zh/hld/part-2-building-blocks/09-pub-sub.md) | 主题、订阅、扇出策略，以及 Google Pub/Sub、Redis、NATS、SNS+SQS 如何在大规模下实现发布/订阅。 |
| 2.10 | [实时通信：WebSocket、SSE 与长轮询](content/zh/hld/part-2-building-blocks/10-real-time-communication.md) | 长轮询、Server-Sent Events、WebSocket、WebRTC 与 MQTT：各自适合什么场景、如何扩展持久连接，以及来自 Discord 和 Slack 的生产经验。 |
| 2.11 | [限流：保护系统不被自己拖垮](content/zh/hld/part-2-building-blocks/11-rate-limiting.md) | 令牌桶、漏桶、固定窗口与滑动窗口、分布式限流，以及 Stripe、Cloudflare、GitHub 如何守护自己的 API。 |
| 2.12 | [服务发现与服务网格：找到服务并与之通信](content/zh/hld/part-2-building-blocks/12-service-discovery-mesh.md) | DNS、客户端发现与服务端发现、健康检查、边车（sidecar）、mTLS，以及 Consul、Envoy、Istio 和 Linkerd 究竟解决了什么问题。 |
| 2.13 | [Blob 与对象存储：存放大块数据](content/zh/hld/part-2-building-blocks/13-blob-object-storage.md) | S3 语义、对象存储的内部实现、分片上传、生命周期策略，以及何时该选 S3、GCS、MinIO 还是文件系统。 |
| 2.14 | [地理空间索引：Geohash、Quadtree、R-tree、S2 与 H3](content/zh/hld/part-2-building-blocks/14-geospatial-indexing.md) | 用空间填充曲线、分层网格和树索引支撑位置查询，并给出 geohash、quadtree、R-tree、S2、H3 之间的选型建议。 |
| 2.15 | [边缘计算（Cloudflare Workers、Lambda@Edge、Deno Deploy）](content/zh/hld/part-2-building-blocks/15-edge-computing.md) | 为边缘设计应用：冷启动、用 Durable Objects 做状态复制、边缘数据库，以及把代码跑在用户附近的边界在哪里。 |

</details>

<details>
<summary><a id="hld-part-3"></a><strong>第 3 部分 · 分布式系统理论</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 准备冲 Senior/Staff 的 SDE2 及以上。如果你还没把线性一致性与可串行化彻底吃透，读这一部分。

让分布式系统真正*分布式*的那套理论：共识（Raft/Paxos）、完整的一致性谱系、2025 年语境下的 CAP/PACELC、逻辑时钟、CRDT、分布式事务（2PC/Saga）、精确一次投递、故障检测、一致性哈希，以及用 Merkle 树做反熵。

| # | 章节 | 内容简介 |
|---|---|---|
| 3.0 | [共识协议：分布式系统如何达成一致](content/zh/hld/part-3-distributed-systems-theory/00-consensus-protocols.md) | Raft 与 Multi-Paxos 详解：主节点选举、日志复制、任期编号下的安全性，以及 etcd、Consul、CockroachDB 为何选择 Raft。 |
| 3.1 | [一致性深入探讨：线性一致性、可串行化，以及两者之间的光谱](content/zh/hld/part-3-distributed-systems-theory/01-consistency-deep-dive.md) | 线性一致性、可串行化、因果一致性与最终一致性的区别，外部一致性，以及如何准确判断你的数据库到底给了你什么保证。 |
| 3.2 | [法定人数与复制：R + W > N 背后的数学](content/zh/hld/part-3-distributed-systems-theory/02-quorums-and-replication.md) | 读写法定人数、Dynamo 风格的复制、松散的法定人数、提示移交，以及法定人数何时能给你线性一致性（以及何时不能）。 |
| 3.3 | [CAP 与 PACELC：那个始终让人困惑的权衡](content/zh/hld/part-3-distributed-systems-theory/03-cap-and-pacelc.md) | CAP 到底说了什么（以及没说什么）、三大常见误解，还有 PACELC 如何补上被漏掉的延迟维度。 |
| 3.4 | [时钟与定序：Lamport 时钟、向量时钟与混合逻辑时钟](content/zh/hld/part-3-distributed-systems-theory/04-clocks-and-ordering.md) | 为什么墙上时钟会撒谎：happens-before 关系、Lamport 时钟、向量时钟、混合逻辑时钟，以及 Google TrueTime。 |
| 3.5 | [CRDT：无冲突复制数据类型](content/zh/hld/part-3-distributed-systems-theory/05-crdts.md) | 基于状态（CvRDT）与基于操作（CmRDT）的 CRDT、G-Counter、PN-Counter、OR-Set、LWW-Register，以及 Yjs、Automerge、Figma 和 Redis 如何使用它们。 |
| 3.6 | [分布式事务：2PC、Saga，以及何时两者都别用](content/zh/hld/part-3-distributed-systems-theory/06-distributed-transactions.md) | 两阶段提交、Percolator、Saga、outbox 模式，以及关于分布式事务的诚实答案：别做，或者付出代价。 |
| 3.7 | [幂等性与精确一次：关于投递保证的老实话](content/zh/hld/part-3-distributed-systems-theory/07-idempotency-exactly-once.md) | 为什么精确一次投递是个神话，幂等键如何让至少一次投递用起来像精确一次，以及 Stripe 和 Kafka 各自是怎么实现的。 |
| 3.8 | [故障检测：判定一个节点已经死亡](content/zh/hld/part-3-distributed-systems-theory/08-failure-detection.md) | 心跳、phi-accrual 检测器、gossip（SWIM），以及为什么“它是死了还是只是慢”没有正确答案。 |
| 3.9 | [一致性哈希：把键分给节点，不必全局重排](content/zh/hld/part-3-distributed-systems-theory/09-consistent-hashing.md) | 哈希环、虚拟节点、有界负载变体、Rendezvous 哈希、Maglev 与 jump 哈希，用于把键分发到不断变化的节点集合上。 |
| 3.10 | [Merkle 树与反熵：低成本保持副本同步](content/zh/hld/part-3-distributed-systems-theory/10-merkle-trees-anti-entropy.md) | Merkle 树、反熵协议、读修复、提示移交，以及 Dynamo、Cassandra 与 Git 如何用哈希树找出彼此的差异。 |

</details>

<details>
<summary><a id="hld-part-4"></a><strong>第 4 部分 · 数据系统</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 任何负责一条数据管道、或要挑数据库的人。

你可能选到的每一种数据系统，以及各自真正适用的场景。存储引擎（B 树 vs LSM）、OLTP vs OLAP、数据仓库/数据湖/湖仓、流处理 vs 批处理、CDC、搜索、时序、图、向量和键值存储。

| # | 章节 | 内容简介 |
|---|---|---|
| 4.0 | [存储引擎：B 树、LSM 树，以及你的数据库为何如此表现](content/zh/hld/part-4-data-systems/00-storage-engines.md) | B 树与 LSM 树存储引擎如何塑造读放大、写放大与空间放大，并结合 InnoDB、PostgreSQL、RocksDB 和 Cassandra 的实例说明。 |
| 4.1 | [OLTP vs OLAP：行式存储、列式存储与负载形态匹配](content/zh/hld/part-4-data-systems/01-oltp-vs-olap.md) | 为什么事务型系统用行式存储、分析型系统用列式存储，并结合 Postgres、MySQL、Redshift、BigQuery、ClickHouse 与 Snowflake 的实例说明。 |
| 4.2 | [数据仓库与数据湖：结构、Schema 与湖仓一体](content/zh/hld/part-4-data-systems/02-data-warehouses-lakes.md) | Redshift、BigQuery、Snowflake、基于 S3 的数据湖，以及由 Delta Lake、Iceberg、Hudi 构成的湖仓一体模式，究竟是怎么拼到一起的。 |
| 4.3 | [流处理与批处理：Lambda、Kappa，以及那场争论的终结](content/zh/hld/part-4-data-systems/03-stream-vs-batch.md) | 用 Spark 和 Hadoop 做批处理、用 Kafka Streams、Flink 和 Spark Streaming 做流处理，以及 Lambda 与 Kappa 架构各自成色如何。 |
| 4.4 | [变更数据捕获：流式输出数据库的内心独白](content/zh/hld/part-4-data-systems/04-change-data-capture.md) | Debezium、Maxwell 与 outbox 模式如何把 WAL 和 binlog 条目变成可靠的事件流，以及每种方案何时才是正确选择。 |
| 4.5 | [搜索系统：倒排索引、BM25 与生产环境中的 Elasticsearch](content/zh/hld/part-4-data-systems/05-search-systems.md) | Elasticsearch、OpenSearch 与 Solr 如何构建倒排索引、用 BM25 打分，以及在大规模下处理分面、相关性调优与分片。 |
| 4.6 | [时序数据库：大规模场景下的指标、事件与数据保留](content/zh/hld/part-4-data-systems/06-time-series-databases.md) | Prometheus、InfluxDB、TimescaleDB 与 VictoriaMetrics 如何靠降采样和保留策略，扛住写密集的时序负载。 |
| 4.7 | [图数据库：属性图、Cypher，以及何时 JOIN 才是问题所在](content/zh/hld/part-4-data-systems/07-graph-databases.md) | Neo4j、Amazon Neptune 和 Dgraph 如何建模关系，以及图查询何时能胜过递归 SQL JOIN。 |
| 4.8 | [向量数据库：嵌入、ANN 索引与 AI 检索层](content/zh/hld/part-4-data-systems/08-vector-databases.md) | Pinecone、Weaviate、Milvus 与 pgvector 如何用 HNSW 和 IVF 近似最近邻索引存储并检索嵌入向量。 |
| 4.9 | [键值存储：Redis、Memcached、DynamoDB，以及如何选对那张哈希表](content/zh/hld/part-4-data-systems/09-key-value-stores.md) | Redis、Memcached、DynamoDB 在持久性、数据模型和扩展方式上的差异，以及各自适用的场景。 |

</details>

<details>
<summary><a id="hld-part-5"></a><strong>第 5 部分 · 架构模式</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 做架构级决策、或主导服务迁移的工程师。

当你设计的东西比单个服务更大时，要在这些架构形态之间做选择：单体 vs 微服务、事件驱动、CQRS、事件溯源、Serverless、BFF、绞杀者模式、六边形/整洁架构、多区域、多租户、基于 CRDT 的应用。

| # | 章节 | 内容简介 |
|---|---|---|
| 5.0 | [单体 vs 微服务：团队拓扑、康威定律与分布式系统税](content/zh/hld/part-5-architecture-patterns/00-monolith-vs-microservices.md) | 什么时候模块化单体胜过微服务、康威定律如何塑造架构，以及分布式系统税到底要你付出什么代价。 |
| 5.1 | [事件驱动架构：事件通知、状态传递与协同](content/zh/hld/part-5-architecture-patterns/01-event-driven-architecture.md) | 事件的三种形态，Kafka 与事件总线如何实现松耦合，以及什么时候协同优于编排。 |
| 5.2 | [CQRS：把读写分开，又不把自己逼疯](content/zh/hld/part-5-architecture-patterns/02-cqrs.md) | 实践中的命令查询职责分离：什么时候该拆开读模型与写模型、如何处理最终一致性，以及什么情况下 CQRS 属于过度设计。 |
| 5.3 | [事件溯源：事件即事实来源](content/zh/hld/part-5-architecture-patterns/03-event-sourcing.md) | 把状态存成只追加的事件日志：重放、投影、快照，以及运行事件溯源系统时的运维现实。 |
| 5.4 | [Serverless：函数、冷启动，以及 FaaS 究竟何时真能省钱](content/zh/hld/part-5-architecture-patterns/04-serverless.md) | AWS Lambda、Google Cloud Functions 与 Azure Functions 实战：冷启动、并发模型，以及 serverless 那份诚实的成本账。 |
| 5.5 | [Backend for Frontend：为每个客户端做好 API 聚合](content/zh/hld/part-5-architecture-patterns/05-backend-for-frontend.md) | 当一个 API 无法同时服务好 Web、移动端和合作伙伴时，BFF 模式给每个客户端一套自己的聚合层。 |
| 5.6 | [绞杀者榕树：不做大爆炸式重写的渐进迁移](content/zh/hld/part-5-architecture-patterns/06-strangler-fig.md) | Martin Fowler 的绞杀者榕树模式：用路由与门面渐进替换遗留系统，以及团队实际执行多年迁移的方式。 |
| 5.7 | [六边形架构与整洁架构：让业务逻辑保持独立](content/zh/hld/part-5-architecture-patterns/07-hexagonal-clean-architecture.md) | 端口与适配器、整洁架构、洋葱架构：如何让领域逻辑保持可测试，并不依赖框架。 |
| 5.8 | [多区域架构：主备、双活与 CRDT](content/zh/hld/part-5-architecture-patterns/08-multi-region-architecture.md) | 设计能扛住区域级故障的系统：DNS 故障切换、主备复制、基于 CRDT 的双活，以及 Cloudflare 的模型。 |
| 5.9 | [多租户：Silo、Pool 与 SaaS 的隔离光谱](content/zh/hld/part-5-architecture-patterns/09-multi-tenancy.md) | 设计在同一套共享基础设施上承载大量租户的 SaaS 平台：隔离级别、吵闹邻居防御、按租户计量，以及何时把鲸鱼租户迁出去。 |
| 5.10 | [CRDT 应用（Yjs、Automerge、本地优先软件）](content/zh/hld/part-5-architecture-patterns/10-crdt-applications.md) | 用 CRDT 设计本地优先的协作软件：Yjs、Automerge、点对点同步，以及从权威中央服务器转向的架构变革。 |

</details>

<details>
<summary><a id="hld-part-6"></a><strong>第 6 部分 · 可靠性与运维</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 任何在值班的人、任何签 SLA 的人、任何付云账单的人。

把“能编译”和“能在长周末凌晨三点稳定运行”区分开的那部分工程：可观测性、SLO、韧性模式、自动扩缩容、部署、混沌工程、故障响应、健康检查、FinOps、平台工程。

| # | 章节 | 内容简介 |
|---|---|---|
| 6.0 | [可观测性：指标、日志、追踪与 OpenTelemetry 标准](content/zh/hld/part-6-reliability-and-operations/00-observability.md) | 可观测性的三大支柱、USE 与 RED 方法论，以及 OpenTelemetry、Prometheus、Grafana 和 Jaeger 在生产中如何配合。 |
| 6.1 | [SLI、SLO、SLA 与错误预算：让可靠性可量化](content/zh/hld/part-6-reliability-and-operations/01-sli-slo-sla-error-budgets.md) | Google SRE 的可靠性框架：该衡量什么、该定什么目标、该承诺什么，以及错误预算如何为功能开发与可靠性工作提供资金。 |
| 6.2 | [韧性模式：超时、重试、熔断器与舱壁隔离](content/zh/hld/part-6-reliability-and-operations/02-resilience-patterns.md) | 让分布式系统免于级联雪崩的防御性模式，从 Hystrix 到现代服务网格实现。 |
| 6.3 | [优雅降级：部分可用胜过完全不可用](content/zh/hld/part-6-reliability-and-operations/03-graceful-degradation.md) | 主动丢弃、特性开关、缓存兜底，以及为了保住整个系统而让某个功能降级背后的产品与工程决策。 |
| 6.4 | [自动扩缩容与容量规划：从 HPA 到预测式扩缩容](content/zh/hld/part-6-reliability-and-operations/04-auto-scaling-capacity.md) | 水平 Pod 自动扩缩容、集群自动扩缩容、预测式扩缩容，以及让系统容量刚好合适又不过度花钱的容量规划算术。 |
| 6.5 | [部署策略：蓝绿、金丝雀、滚动与特性开关](content/zh/hld/part-6-reliability-and-operations/05-deployment-strategies.md) | 如何用蓝绿、滚动、金丝雀和渐进式交付安全地上线变更，以及特性开关与 LaunchDarkly 一类工具在其中扮演的角色。 |
| 6.6 | [混沌工程：故意把系统搞坏](content/zh/hld/part-6-reliability-and-operations/06-chaos-engineering.md) | Netflix 的 Chaos Monkey、混沌原则，以及如何在不把值班同事逼到请病假的前提下，开展演练日和故障注入实验。 |
| 6.7 | [事件管理：从检测到无指责复盘](content/zh/hld/part-6-reliability-and-operations/07-incident-management.md) | 值班、事件指挥、严重级别、沟通，以及如何开展真正能改变系统、而不是追责于人的事后复盘。 |
| 6.8 | [健康检查与就绪：如实回答服务是否在线](content/zh/hld/part-6-reliability-and-operations/08-health-checks-readiness.md) | 存活、就绪与启动探针，深层检查与浅层检查的差别，以及为什么糟糕的健康检查比糟糕的代码造成更多故障。 |
| 6.9 | [成本优化与 FinOps](content/zh/hld/part-6-reliability-and-operations/09-cost-optimization-finops.md) | 用 FinOps 降低云账单而不牺牲可靠性：Spot 实例、预留容量、自动扩缩容、存储分层，以及单位经济学的思维方式。 |
| 6.10 | [平台工程：IDP、黄金路径与开发者体验](content/zh/hld/part-6-reliability-and-operations/10-platform-engineering.md) | 把平台当产品做：用 Backstage、黄金路径，以及 DORA/SPACE 指标搭出能真正提升开发者生产力的内部开发者平台。 |

</details>

<details>
<summary><a id="hld-part-7"></a><strong>第 7 部分 · 规模化安全</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 高级/Staff 工程师、平台团队、与安全相邻的建设者。

面向真实系统的安全架构，不是一张 CISSP 背诵卡：认证 vs 授权（AuthN vs AuthZ）、OAuth2/OIDC、JWT（以及为什么你大概不该用它）、mTLS、密钥管理、DDoS/WAF、合规（GDPR/DPDP/CCPA）、软件供应链、隐私保护系统、后量子密码学。

| # | 章节 | 内容简介 |
|---|---|---|
| 7.0 | [认证 vs 授权：权限与访问模型](content/zh/hld/part-7-security-at-scale/00-authn-authz.md) | AuthN 与 AuthZ 的区别、会话认证与令牌认证，以及 RBAC、ABAC、ReBAC 几种访问控制模型，附 SpiceDB、OpenFGA 与 AWS IAM 的实例。 |
| 7.1 | [OAuth 2.0 与 OpenID Connect：正确实现委派授权与身份](content/zh/hld/part-7-security-at-scale/01-oauth2-oidc.md) | OAuth 2.0 授权框架、OIDC 身份层，以及已成为 Web 与移动端现代标准的授权码 + PKCE 流程。 |
| 7.2 | [JWT 深入解析：签名令牌、声明与吊销难题](content/zh/hld/part-7-security-at-scale/02-jwt-deep-dive.md) | JSON Web Token 的工作机制：JWS 签名、JWE 加密、声明校验、密钥轮换，以及无状态认证的权衡。 |
| 7.3 | [mTLS 与服务间认证：SPIFFE、服务网格与零信任](content/zh/hld/part-7-security-at-scale/03-mtls-service-auth.md) | 双向 TLS、SPIFFE/SPIRE，以及 Istio、Linkerd 这类服务网格，如何在不依赖长期凭据的前提下完成服务认证。 |
| 7.4 | [密钥管理：Vault、KMS，以及配置文件里密钥的终结](content/zh/hld/part-7-security-at-scale/04-secrets-management.md) | 用 Vault、AWS Secrets Manager、KMS 信封加密与动态密钥，管理 API key、密码和证书。 |
| 7.5 | [DDoS 防护与 WAF：抵御流量型与应用层攻击](content/zh/hld/part-7-security-at-scale/05-ddos-waf.md) | 借助 Cloudflare、AWS Shield 与 WAF 防御 L3/L4 和 L7 的 DDoS；限流、机器人管理与 OWASP Top 10。 |
| 7.6 | [数据驻留与合规架构（GDPR、DPDP、CCPA、删除权）](content/zh/hld/part-7-security-at-scale/06-data-residency-compliance.md) | 为 GDPR、DPDP、CCPA 与 LGPD 设计跨司法辖区的系统：数据分类、区域孤岛、加密销毁，以及可审计的删除。 |
| 7.7 | [供应链安全：SBOM、SLSA、Sigstore，以及如何防御 xz-utils 式攻击](content/zh/hld/part-7-security-at-scale/07-supply-chain-security.md) | 用 SBOM、SLSA 来源证明、Sigstore 签名、准入策略保护软件供应链，以及 xz-utils 与 SolarWinds 事件留下的教训。 |
| 7.8 | [隐私保护系统（差分隐私、联邦学习）](content/zh/hld/part-7-security-at-scale/08-privacy-preserving-systems.md) | 从设计层面保护用户数据的系统：差分隐私、联邦学习、安全聚合，以及同态加密入门。 |
| 7.9 | [后量子密码学：迁移到 ML-KEM、ML-DSA 与密码敏捷的未来](content/zh/hld/part-7-security-at-scale/09-post-quantum-crypto.md) | 为什么“先存后解”让 PQC 变得紧迫、NIST 在 2024 年标准化了什么，以及如何把生产环境的 TLS 与长期密钥迁移到混合后量子密码学。 |

</details>

<details>
<summary><a id="hld-part-8"></a><strong>第 8 部分 · 案例研究</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 面试备考者、在构建相邻系统的工程师、任何靠实例学得最快的人。

端到端的系统设计，每一篇都遵循统一结构：需求（功能 + 非功能）、量级估算数字、高层架构、数据模型、深入探讨的组件、可扩展性与可靠性考量，以及真实案例参考。下面按主题分组，方便查找。

#### 核心构件

面试经典题。每章一个构件，从零开始设计。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.0 | [设计一个 URL 短链接服务（TinyURL / bit.ly）](content/zh/hld/part-8-case-studies/00-url-shortener.md) | 一份面试级的 URL 短链接服务演练：容量估算、短码生成、热键缓存，以及一条从不阻塞重定向的分析流水线。 |
| 8.1 | [设计 Pastebin（粘贴分享服务）](content/zh/hld/part-8-case-studies/01-pastebin.md) | 面向面试的 Pastebin 风格文本分享服务全流程推演：blob 与元数据的存储拆分、基于 TTL 的过期流水线、语法高亮的放置位置，以及滥用防护。 |
| 8.2 | [设计一个分布式限流器](content/zh/hld/part-8-case-studies/02-rate-limiter.md) | 分布式 API 限流器的面试级完整拆解：算法选型、Redis Lua 原子性、本地加全局的两层同步，以及失效开放（fail-open）的故障模式。 |
| 8.3 | [设计一个分布式键值存储（Dynamo / Cassandra / Riak）](content/zh/hld/part-8-case-studies/03-key-value-store.md) | 用一致性哈希、法定人数复制、gossip 成员管理、提示移交和 Merkle 树反熵修复来设计一个分布式 KV 存储。 |
| 8.4 | [设计一个通知系统（大规模推送、短信与邮件）](content/zh/hld/part-8-case-studies/04-notification-system.md) | 一道面试级的多渠道通知平台设计题：扇出架构、APNs/FCM 接入、带死信队列的重试，以及设备令牌的生命周期管理。 |
| 8.5 | [设计聊天系统（WhatsApp / Messenger / Signal）](content/zh/hld/part-8-case-studies/05-chat-system.md) | WhatsApp 量级的 1:1 与小群聊资深级设计：5 亿并发连接、消息定序、端到端加密，以及存储模型的权衡。 |
| 8.6 | [设计社交媒体信息流（Twitter / Instagram / LinkedIn）](content/zh/hld/part-8-case-studies/06-social-media-feed.md) | 扇出架构、推拉混合、ML 排序流水线，以及 Twitter/X 规模下的名人问题。 |
| 8.7 | [设计照片分享服务（Instagram）](content/zh/hld/part-8-case-studies/07-photo-sharing.md) | 设计 Instagram 规模的照片分享服务：上传流水线、转码、多分辨率图片分发、带源站保护的 CDN，以及与信息流的集成。 |
| 8.8 | [设计一个网络爬虫（Googlebot 风格）](content/zh/hld/part-8-case-studies/08-web-crawler.md) | 设计一个分布式网络爬虫：URL 前沿、礼貌策略、内容去重、robots.txt 合规，以及十亿级规模下由 Bloom filter 支撑的 URL 去重。 |
| 8.9 | [设计搜索自动补全（Typeahead 输入联想）](content/zh/hld/part-8-case-studies/09-search-autocomplete.md) | 在 Google 规模下设计一个低延迟的自动补全系统：trie、top-K 预计算、实时趋势叠加层与多级缓存。 |
| 8.16 | [设计一个分布式缓存（Memcached / Redis Cluster）](content/zh/hld/part-8-case-studies/16-distributed-cache.md) | 设计一个 Memcached 或 Redis 风格的分布式缓存：一致性哈希、淘汰策略、复制、客户端分片与热键缓解。 |
| 8.25 | [设计一个邻近服务（附近好友 / Yelp）](content/zh/hld/part-8-case-studies/25-proximity-service.md) | 为 1 亿用户、100 万并发位置分享会话设计邻近服务：geohash/H3/S2 的权衡、Redis geoset、包围盒查询扇出、隐私保护，以及大型活动期间的热点处理。 |
| 8.26 | [设计实时排行榜](content/zh/hld/part-8-case-studies/26-leaderboard.md) | 为 1000 万玩家、每秒 10 万次分数更新设计实时排行榜：并列裁决、时间窗口视图、好友榜，以及长尾玩家的近似名次。 |
| 8.27 | [设计一个唯一 ID 生成器（Snowflake、ULID、TSID、UUIDv7）](content/zh/hld/part-8-case-studies/27-unique-id-generator.md) | 设计一个分布式 ID 生成器：每秒产出 1000 万个 64 位 ID，近似有序，能扛住时钟偏移，并在 Snowflake、ULID、TSID 与 UUIDv7 之间做出四维权衡。 |
| 8.29 | [设计分布式任务调度器（Airflow / Temporal / 分布式 Cron）](content/zh/hld/part-8-case-studies/29-job-scheduler.md) | 设计一个调度器：10 万个注册任务、每秒 1 万次执行，保证恰好一次执行，支持上万节点的 DAG、漏跑/迟跑策略，以及调度器的优雅升级。 |

#### 消费级产品

由多个构件搭起来的终端用户产品。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.10 | [设计视频流媒体服务（YouTube / Twitch / TikTok）](content/zh/hld/part-8-case-studies/10-video-streaming.md) | 设计一个 UGC 视频平台，从上传到自适应码率播放：转码流水线、HLS/DASH/CMAF 打包、带 ISP 对等互联的 CDN 分发，以及直播接入架构。 |
| 8.11 | [设计 Netflix（端到端）](content/zh/hld/part-8-case-studies/11-netflix.md) | 完整走一遍 Netflix 的架构：微服务、Open Connect CDN、按片编码、Cassandra + EVCache、韧性模式与混沌工程。 |
| 8.12 | [设计一个网约车服务（Uber / Lyft）](content/zh/hld/part-8-case-studies/12-ride-hailing.md) | 面向面试的 Uber 量级网约车系统全流程讲解：H3 地理空间索引、实时位置摄入、批量二分图匹配、动态加价与行程编排。 |
| 8.13 | [设计 Google Maps（路线规划与瓦片渲染）](content/zh/hld/part-8-case-studies/13-google-maps.md) | 设计行星级地图产品：瓦片渲染、基于收缩层次的最短路由、GNN 预测 ETA，以及离线地图。 |
| 8.14 | [设计文件同步服务（Dropbox / Google Drive）](content/zh/hld/part-8-case-studies/14-file-sync.md) | 设计一个 Dropbox 风格的文件同步服务：块级去重、增量同步、冲突解决、版本管理，以及客户端与服务端的状态协调。 |
| 8.15 | [设计协同编辑系统（Google Docs / Figma / Notion）](content/zh/hld/part-8-case-studies/15-collaborative-editing.md) | 面向 10 万以上并发编辑者的实时协同编辑资深级设计：OT 与 CRDT、在线状态广播、离线同步与版本历史。 |
| 8.18 | [设计一个票务系统（BookMyShow / Ticketmaster）](content/zh/hld/part-8-case-studies/18-ticketing-system.md) | 面向高并发票务的面试级完整拆解：用 Redis SETNX 做座位锁定、基于 Saga 的支付编排、虚拟等候室，以及在每年 5 亿张票的规模下抵御机器人的防线。 |
| 8.21 | [设计一个外卖配送服务（DoorDash / Swiggy）](content/zh/hld/part-8-case-studies/21-food-delivery.md) | 设计一个三边外卖市场：带合并与改派的调度、复合 ETA 预测，以及骑手、商家、顾客三方的协同。 |
| 8.28 | [设计酒店预订系统（Booking.com / Airbnb）](content/zh/hld/part-8-case-studies/28-hotel-reservation.md) | 面向资深工程师的酒店预订系统设计：搜索与预订拆分、用 PostgreSQL 排他约束在数据库层杜绝日期区间重复售卖、Temporal saga 编排，以及基于 CDC 的可用性索引在 85K QPS 下的落地。 |
| 8.40 | [设计一个约会应用（Tinder / Hinge / Bumble）](content/zh/hld/part-8-case-studies/40-dating-app.md) | 为 1 亿 MAU 设计约会应用：每天 15 亿次滑动、卡片加载 <50 ms——双塔推荐、地理过滤、互相匹配检测，以及大规模安全治理。 |
| 8.43 | [设计视频会议系统（Zoom / Google Meet）](content/zh/hld/part-8-case-studies/43-video-conferencing.md) | 为 50 万场同时进行的会议和 1000 万并发参会者设计视频会议平台，音频 <150 ms、视频 <500 ms：SFU vs MCU vs P2P、simulcast 与 SVC、带宽估计、区域媒体路由与实时录制。 |
| 8.44 | [设计 Gmail 规模的邮件服务（18 亿用户，每天 3000 亿封邮件）](content/zh/hld/part-8-case-studies/44-gmail-scale-email.md) | 设计支撑 18 亿用户、每天 3000 亿封邮件的全球邮件服务：SMTP 接入口、垃圾邮件级联流水线、按用户分片的搜索、RFC 5322 会话串接，以及带内容寻址附件去重的 EB 级分层存储。 |
| 8.45 | [大规模场景下的直播评论系统设计（FB Live / YouTube Live / Twitch Chat）](content/zh/hld/part-8-case-studies/45-live-comments.md) | 为单场直播设计一套支撑 1000 万并发观众、10 万评论者的直播评论系统：增量批处理扇出、发布前审核，以及明星直播的热点问题。 |
| 8.47 | [设计一个健身追踪服务（Strava / MapMyRun）](content/zh/hld/part-8-case-studies/47-strava-fitness.md) | 为 1.95 亿+ 用户设计健身追踪服务：GPS 摄入、H3 预筛加 DTW 的两阶段赛段匹配、Kafka 驱动的排行榜，以及隐私安全的行星级热力图。 |

#### 金融系统

正确性、审计与监管优先的系统。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.19 | [设计支付系统（Stripe / PayPal）](content/zh/hld/part-8-case-studies/19-payment-system.md) | 设计一个支付系统：复式记账账本、幂等键、由 Saga 编排的跨服务流程，以及塑造每一个决策的合规约束。 |
| 8.20 | [设计证券交易所（撮合引擎）](content/zh/hld/part-8-case-studies/20-stock-exchange.md) | 设计一台确定性、低延迟的撮合引擎：FIFO 订单簿、价格-时间优先、组播行情分发，以及托管机房带来的现实约束。 |
| 8.41 | [设计在线拍卖系统（eBay / Catawiki）](content/zh/hld/part-8-case-studies/41-online-auction.md) | 为 1 亿条在售标的和 1000 万并发竞拍者设计在线拍卖系统：用 Redis Lua CAS 在每秒 100 万次峰值下原子接受出价，并处理代理出价、狙击延时与 WebSocket 扇出。 |
| 8.46 | [设计一个欺诈检测系统（Stripe Radar / PayPal / Feedzai）](content/zh/hld/part-8-case-studies/46-fraud-detection.md) | 设计一个实时欺诈检测服务，用规则 + ML 级联、在线/离线特征存储、图团伙检测和分析师反馈闭环，在 100 ms P99 内为每秒数百万事件评分。 |
| 8.54 | [设计一个券商交易平台（Robinhood / E*TRADE / Interactive Brokers）](content/zh/hld/part-8-case-studies/54-brokerage-trading.md) | 为 3000 万用户设计零售券商：订单路由、按代码分频道的行情扇出、碎股聚合、税批核算，以及七年监管审计。 |

#### AI 系统

难点在推理、检索或智能体行为的系统。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.17 | [设计一个推荐系统（Netflix / YouTube / TikTok）](content/zh/hld/part-8-case-studies/17-recommendation-system.md) | 设计一个两阶段推荐系统：候选生成、排序、协同过滤、基于内容的特征、特征存储，以及冷启动处理。 |
| 8.30 | [设计 ChatGPT（大规模对话式 AI）](content/zh/hld/part-8-case-studies/30-chatgpt-conversational-ai.md) | 为 9 亿周活用户设计 ChatGPT：多租户 LLM 服务、会话状态架构、SSE 流式输出、用户级记忆、安全审核与多区域部署。 |
| 8.31 | [设计企业级 RAG 系统](content/zh/hld/part-8-case-studies/31-enterprise-rag.md) | 为 1,000 个租户、每租户 1,000 万份文档、每租户 100 QPS 设计一个多租户企业 RAG 平台：摄取、混合检索、重排、引用、访问控制感知的检索、新鲜度与租户隔离。 |
| 8.32 | [设计一个编程 Agent（Claude Code / GitHub Copilot / Cursor）](content/zh/hld/part-8-case-studies/32-coding-agent.md) | 设计一个编程 Agent，覆盖自动补全、对话与自主循环三种模式，服务 100 万并发会话，包含仓库索引、沙箱化工具调用与流式 diff 应用。 |
| 8.33 | [设计 Perplexity（带引用的 AI 搜索）](content/zh/hld/part-8-case-studies/33-perplexity-ai-search.md) | 为 5000 万 MAU、峰值 5k QPS、答案延迟低于 2 秒且带行内引用的场景设计一个 AI 搜索引擎：查询改写、来源检索、引用接地、流式输出、时效性与事实核查后处理。 |
| 8.34 | [设计语音智能体（Alexa / Siri 级实时交互）](content/zh/hld/part-8-case-studies/34-voice-agent.md) | 为 1 亿台设备、5 万路并发对话和低于 700 ms 的轮次延迟设计实时语音智能体：流式 ASR、LLM 轮次切换、流式 TTS、WebRTC 传输与打断（barge-in）。 |
| 8.35 | [设计一个大规模内容审核系统](content/zh/hld/part-8-case-studies/35-content-moderation-at-scale.md) | 为每天 5 亿条帖子设计内容审核：发布前延迟 <200 ms，0.5% 流量转人工，多模态（文本 + 图片 + 视频）：分类器级联、审核员工作流、申诉、对抗鲁棒性。 |
| 8.36 | [为 LLM 应用设计语义缓存](content/zh/hld/part-8-case-studies/36-semantic-cache.md) | 在 10k QPS、70%+ 命中率、查询延迟 <10 ms 的条件下，为 LLM prompt 设计一个基于嵌入相似度的缓存：相似度阈值标定、源文档变更时的失效、按租户隔离、误命中成本核算。 |
| 8.37 | [设计模型路由器与网关（OpenRouter / LiteLLM）](content/zh/hld/part-8-case-studies/37-model-router-gateway.md) | 设计一个网关：在 50+ 个模型、10 家供应商之上路由 20k QPS，路由开销低于 30 ms：成本/延迟/质量路由策略、供应商故障切换、流式透传、按租户配额、成本归因。 |

#### 基础设施平台

其他工程师在其之上构建的平台。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.22 | [设计指标流水线（Prometheus / InfluxDB / Thanos）](content/zh/hld/part-8-case-studies/22-metrics-pipeline.md) | 设计一条时间序列指标流水线：高基数摄取、用 Thanos 或 Cortex 做跨集群聚合、告警，以及为长期保留做降采样。 |
| 8.23 | [设计广告点击聚合（实时流处理）](content/zh/hld/part-8-case-studies/23-ad-click-aggregation.md) | 设计一个广告点击聚合系统：Kafka + Flink 上的精确一次语义、实时欺诈检测，以及低延迟看板。 |
| 8.24 | [设计一个日志平台（ELK / Loki / Splunk）](content/zh/hld/part-8-case-studies/24-logging-platform.md) | 设计一个日志平台：大规模摄取、基于索引与基于标签的存储（Elastic 与 Loki）、分层保留，以及用 BM25 做全文搜索。 |
| 8.38 | [设计一个特性开关服务（LaunchDarkly / Harness FME / Unleash）](content/zh/hld/part-8-case-studies/38-feature-flag-service.md) | 为每天 20 万亿次求值设计一个特性开关与实验平台，SDK 侧延迟低至亚毫秒，配置以流式分发，紧急开关在 60 秒内完成全球传播。 |
| 8.39 | [设计一个 DNS 服务（Cloudflare 1.1.1.1 / Google 8.8.8.8）](content/zh/hld/part-8-case-studies/39-dns-service.md) | 设计一个公共服务递归 DNS 解析器：从 300+ 个 anycast POP 为全球每天数万亿次查询提供服务，P99 低于 20 ms；涵盖 UDP/TCP/DoT/DoH/DoQ、DNSSEC 验证、缓存设计、限流与 DDoS 生存能力。 |
| 8.42 | [设计多租户 SaaS 平台](content/zh/hld/part-8-case-studies/42-multi-tenant-saas.md) | 设计一个承载 5 万租户的多租户 SaaS 平台，具备按租户分级的 SLA、按用量计量计费、吵闹邻居隔离，以及零跨租户数据泄露。 |
| 8.48 | [设计在线评测系统（LeetCode / Codeforces / HackerEarth）](content/zh/hld/part-8-case-studies/48-online-judge.md) | 为 100 万用户、峰值 10 万次提交/小时、判定 p99 < 15s 设计在线评测系统：Firecracker 沙箱、竞赛优先级排队、seccomp + cgroups 隔离，以及 MOSS 风格的查重。 |
| 8.49 | [设计一个价格追踪服务（CamelCamelCamel / Honey / Keepa）](content/zh/hld/part-8-case-studies/49-price-tracking.md) | 设计一个价格追踪服务：按优先级调度监控 1 亿个商品 URL，用差分比对把告警推给 1000 万订阅者，保留 2 年历史数据，并在零售商主动封堵的情况下礼貌地轮换代理池。 |
| 8.50 | [设计一个大规模 API 网关（Kong / AWS API Gateway / Apigee / Envoy）](content/zh/hld/part-8-case-studies/50-api-gateway.md) | 设计一个每实例承载 10 万 RPS、p99 额外开销低于 5 ms、后端接 1 万个上游服务的 API 网关：路由基数树、本地加全局限流、mTLS 卸载、无损配置热加载，以及熔断。 |
| 8.51 | [设计一个 CI/CD 平台（GitHub Actions / GitLab CI / CircleCI）](content/zh/hld/part-8-case-studies/51-cicd-platform.md) | 为 10 万个组织、每天 1000 万次工作流运行设计一个 CI/CD 平台：YAML DAG 执行、临时 Runner 池（Firecracker）、内容寻址产物、依赖缓存、密钥注入，以及 4 万并发 Runner 下的多租户隔离。 |
| 8.52 | [设计可观测性平台（Datadog / New Relic / Honeycomb）](content/zh/hld/part-8-case-studies/52-observability-platform.md) | 为 1000 万台主机、每秒 10 亿事件设计统一的可观测性平台：跨指标、日志、追踪的 OTLP 摄入、基数控制、追踪与日志关联、多租户隔离，以及 10 万名并发分析师的查询延迟。 |
| 8.53 | [设计搜索引擎（Google 规模 / Brave Search）](content/zh/hld/part-8-case-studies/53-search-engine.md) | 设计一个覆盖 100 亿文档、承载 10 万 QPS、p99 低于 200 ms 的 Web 级搜索引擎：分片倒排索引、BM25 + PageRank + 神经重排序级联、查询理解，以及时新性管道。 |
| 8.55 | [设计频道级规模的聊天系统（Discord / Slack）](content/zh/hld/part-8-case-studies/55-channel-scale-chat.md) | 为 10 万+ 成员的频道设计频道级规模聊天：pub/sub 扇出、热路径上的 RBAC、在线状态聚合，以及工作区搜索。 |

</details>

<details>
<summary><a id="hld-part-9"></a><strong>第 9 部分 · AI 与 ML 系统设计</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 任何基于 LLM、智能体或生产级 ML 做构建的人，或在它们周边做构建的人。

现代 AI 系统架构，用与第 3 部分同样的严谨度处理：LLM 服务、RAG、向量搜索、智能体架构、多智能体编排、LLM 评估、LLMOps、成本优化、安全、ML 基础、特征存储、推荐系统、实时 AI、多模态，以及支撑这一切的数据基础设施。

| # | 章节 | 内容简介 |
|---|---|---|
| 9.0 | [LLM 服务架构（vLLM、TGI、TensorRT-LLM）](content/zh/hld/part-9-ai-ml-system-design/00-llm-serving-architecture.md) | 设计一套生产级 LLM 推理栈：连续批处理、分页注意力、KV 缓存管理与多租户 GPU 调度。 |
| 9.1 | [RAG 流水线（检索增强生成）](content/zh/hld/part-9-ai-ml-system-design/01-rag-pipelines.md) | 设计生产级 RAG：分块、嵌入模型、稠密加稀疏的混合检索、重排序，以及让整条链路保持诚实的评估闭环。 |
| 9.2 | [大规模向量检索（HNSW、IVF-PQ、DiskANN）](content/zh/hld/part-9-ai-ml-system-design/02-vector-search-at-scale.md) | 设计十亿级向量检索：HNSW、IVF-PQ 与 DiskANN 索引、乘积量化、BM25 与向量的混合检索，以及分片策略。 |
| 9.3 | [AI Agent 架构（ReAct、反思、规划、工具调用、记忆）](content/zh/hld/part-9-ai-ml-system-design/03-ai-agent-architectures.md) | 把 LLM 变成 agent 的几套经典模式：ReAct 的思考-行动-观察循环、反思与自我批评、规划器-执行器分解、工具调用与函数调用，以及 agent 如何管理短期与长期记忆。 |
| 9.4 | [多智能体编排（LangGraph、OpenAI Agents SDK、AutoGen、Swarm）](content/zh/hld/part-9-ai-ml-system-design/04-multi-agent-orchestration.md) | 把多个智能体组合成一个可靠的系统：编排者-工作者拓扑、交接与委派、共享记忆、并行扇出，以及智能体图的失效模式。 |
| 9.5 | [LLM 评估与可观测性（Ragas、LangSmith、TruLens、LLM-as-Judge）](content/zh/hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md) | 如何在 LLM 系统上线前后做评估：金标准数据集、无参考答案的指标、LLM 评审、持续评估流水线，以及生产 LLM 的可观测性技术栈。 |
| 9.6 | [LLMOps 与提示词工程（版本管理、护栏、红队测试）](content/zh/hld/part-9-ai-ml-system-design/06-llmops-prompt-engineering.md) | 把 LLM 功能交付上线的运维面：提示词即代码、版本管理、回滚、提示词 A/B 测试、结构化输出，以及上线前的红队测试。 |
| 9.7 | [LLM 成本优化（语义缓存、模型路由、级联、提示缓存）](content/zh/hld/part-9-ai-ml-system-design/07-llm-cost-optimization.md) | 生产环境 LLM 的成本工程工具箱：语义缓存、模型路由、先小后大的级联、提示缓存（Anthropic、OpenAI），以及决定单请求毛利的那套单位经济账。 |
| 9.8 | [LLM 安全与护栏（OWASP LLM Top 10、提示词注入、PII、越狱）](content/zh/hld/part-9-ai-ml-system-design/08-llm-safety-guardrails.md) | LLM 应用的安全工程面：OWASP LLM Top 10、提示词注入防御、PII 脱敏、越狱围堵，以及面向公众的智能体该怎么做纵深防御。 |
| 9.9 | [ML 系统设计基础](content/zh/hld/part-9-ai-ml-system-design/09-ml-system-design-fundamentals.md) | 所有现代 AI 产品都压在这套经典 ML 系统骨架上：候选生成、排序、双塔嵌入、离线与在线特征一致性，以及训练-服务偏差问题。 |
| 9.10 | [特征存储与模型服务（Feast、Tecton、KServe、BentoML、MLflow）](content/zh/hld/part-9-ai-ml-system-design/10-feature-stores-model-serving.md) | 让机器学习真正能交付的基础设施：在线与离线特征存储、模型注册表、模型服务器、影子部署，以及一个已训练模型周围的生产生命周期。 |
| 9.11 | [推荐系统深入剖析（DLRM、双塔、向量召回、冷启动）](content/zh/hld/part-9-ai-ml-system-design/11-recommendation-systems.md) | 现代推荐系统端到端是怎么运转的：在 embedding 上做 ANN 候选召回、DLRM 式排序、探索与利用、冷启动处理，以及一套让指标不失真的评估闭环。 |
| 9.12 | [实时 AI 与语音代理（流式推理、WebRTC、LiveKit、Deepgram）](content/zh/hld/part-9-ai-ml-system-design/12-realtime-ai-voice-agents.md) | 设计亚秒级语音代理：流式 ASR、低延迟 LLM 推理、流式 TTS、WebRTC 传输、打断处理，以及端到端延迟预算。 |
| 9.13 | [多模态 AI 系统（CLIP、Whisper、LayoutLM、文档 AI）](content/zh/hld/part-9-ai-ml-system-design/13-multimodal-ai-systems.md) | 设计能摄入图像、音频、视频与文档的系统：用于跨模态检索的 CLIP 式 embedding、Whisper 流水线、OCR 加版式模型，以及非结构化数据的存储架构。 |
| 9.14 | [面向 AI 的数据基础设施（嵌入管道、分块、非结构化 ETL、MCP）](content/zh/hld/part-9-ai-ml-system-design/14-data-infrastructure-for-ai.md) | 支撑 AI 系统的数据面：数据源连接器、分块策略、大规模嵌入、元数据 schema、新鲜度，以及作为标准接口的模型上下文协议。 |

</details>

<details>
<summary><a id="hld-part-10"></a><strong>第 10 部分 · 新兴模式</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 在思考 2026 年之后问题的 Staff+ 工程师和架构师。

与书中其他内容并列的前瞻性主题。

| # | 章节 | 内容简介 |
|---|---|---|
| 10.0 | [绿色计算（碳感知调度、PUE、可持续系统）](content/zh/hld/part-10-emerging-patterns/00-green-computing.md) | 设计把碳纳入考量的系统：碳感知调度、PUE、可再生能源数据中心，以及绿色软件基金会的实践。 |

</details>

<details>
<summary><a id="hld-part-11"></a><strong>第 11 部分 · 面试框架</strong></summary>

**级别：** 中级到高级 &nbsp;·&nbsp; **适合：** 任何准备系统设计面试、或坐在面试官那一侧的人。

如何主持一场 45 分钟的系统设计面试，白板两侧的视角都讲。对比 RESHADED / PEDALS / ADEPT 三套框架，讲解需求界定、画图、权衡表述、各公司的风格差异，以及 Staff 级别的 RFC/设计文档写作。

| # | 章节 | 内容简介 |
|---|---|---|
| 11.0 | [主流面试框架对比（RESHADED、PEDALS、ADEPT）](content/zh/hld/part-11-interview-framework/00-frameworks-compared.md) | 对比主流的系统设计面试框架，并选定一个贯穿你每一场面试的框架。 |
| 11.1 | [需求界定：功能性需求、非功能性需求与 MoSCoW](content/zh/hld/part-11-interview-framework/01-requirements-scoping.md) | 掌握系统设计面试的前五分钟：功能性需求与非功能性需求的区分、MoSCoW 优先级排序，以及如何给范围设定时间盒。 |
| 11.2 | [系统设计面试中的画图能力](content/zh/hld/part-11-interview-framework/02-diagramming-skills.md) | 画出面试官看得懂的板书与在线架构图：统一的图例约定、流向规则，以及业界更偏爱的工具（Excalidraw、Miro）。 |
| 11.3 | [权衡表达：把“看情况”说清楚](content/zh/hld/part-11-interview-framework/03-trade-off-articulation.md) | 学会像资深工程师那样把设计权衡说出口：点明权衡维度、说明它依赖什么条件、然后果断做出选择。 |
| 11.4 | [各家公司的面试风格差异（Amazon、Google、Meta、Netflix）](content/zh/hld/part-11-interview-framework/04-company-specific-flavors.md) | 同一道系统设计题在 Amazon、Google、Meta、Netflix 会呈现不同面貌：领导力准则、规模侧重、产品感，以及简洁性。 |
| 11.5 | [设计文档撰写：RFC、ADR 与 Staff 工程师的书面产出](content/zh/hld/part-11-interview-framework/05-design-doc-authoring.md) | 如何撰写 ADR、设计文档与 RFC，让它们推动对齐、记录决策，并展现 Staff+ 级别的工程判断力。 |

</details>

### 权衡库

那些反复出现的设计问题，每一个都在专门的页面上作答：一张决策流程图、一张对比表、何时选哪一边、真实案例与引用。书中各章都会链到这里。

<details>
<summary><strong>全部权衡页面</strong> &nbsp;—&nbsp; 一致性、存储、API、实时、扩展、事务、部署</summary>

| # | 权衡点 | 怎么选 |
|---|---|---|
| 1 | [强一致性 vs 最终一致性](content/zh/hld/trade-offs/01-strong-vs-eventual-consistency.md) | 速查：什么时候该选线性一致性，什么时候最终一致就够了，以及大多数生产系统实际在跑的“会话保证”混合方案。 |
| 2 | [ACID 与 BASE](content/zh/hld/trade-offs/02-acid-vs-base.md) | ACID 事务何时值得付出协调成本，BASE 何时是唯一还能扩展的选择，以及为什么多数系统两者并用。 |
| 3 | [SQL 与 NoSQL](content/zh/hld/trade-offs/03-sql-vs-nosql.md) | 什么时候关系型胜出、什么时候非关系型胜出，以及为什么到 2026 年“SQL 不能扩展”这个说法几乎总是错的。 |
| 4 | [延迟 vs 吞吐量](content/zh/hld/trade-offs/04-latency-vs-throughput.md) | 什么时候该为单个请求做优化，什么时候该为整体做优化，以及为什么批处理存在一个不那么明显的天花板。 |
| 5 | [CAP 与 PACELC 的落地应用](content/zh/hld/trade-offs/05-cap-and-pacelc.md) | 在分区下选 CP 还是 AP 的决策指南，以及为什么 PACELC 的 else 子句（延迟与一致性之争）才是你每一次请求都要面对的选择。 |
| 6 | [缓存策略：旁路缓存 vs 写穿透 vs 写回](content/zh/hld/trade-offs/06-cache-strategies.md) | 一份挑选缓存与数据库协作模式的决策指南：你能得到什么、会失去什么，以及为什么大多数系统最终都用混合方案。 |
| 7 | [批处理与流处理](content/zh/hld/trade-offs/07-batch-vs-stream.md) | 在离散作业运行与持续流动之间做选择的决策指南，锚定在真正重要的新鲜度 SLO 上。 |
| 8 | [负载均衡器 vs 反向代理 vs API 网关](content/zh/hld/trade-offs/08-load-balancer-vs-proxy-vs-gateway.md) | 根据流量层级、协议与策略需求，在 L4 负载均衡器、L7 反向代理和 API 网关之间做选择的决策指南。 |
| 9 | [REST vs gRPC vs GraphQL](content/zh/hld/trade-offs/09-rest-vs-grpc-vs-graphql.md) | 根据调用方身份、缓存需求与负载结构来选择合适 API 协议的决策指南。 |
| 10 | [轮询 vs 长轮询 vs SSE vs WebSockets vs Webhooks](content/zh/hld/trade-offs/10-polling-vs-websockets.md) | 挑选合适实时模式的决策指南。五种方案，各自适配不同的场景；选错就意味着漏掉更新、浪费带宽，或者背上运维负担。 |
| 11 | [限流算法：令牌桶 vs 滑动窗口](content/zh/hld/trade-offs/11-rate-limiting-algorithms.md) | 挑选限流算法的决策指南。令牌桶容忍突发；滑动窗口保证精确；混合路径把两者叠起来。 |
| 12 | [乐观并发控制 vs 悲观并发控制](content/zh/hld/trade-offs/12-optimistic-vs-pessimistic-locking.md) | 在提交时校验与先加锁之间做选择的决策指南。答案取决于冲突率、重试成本和副作用。 |
| 13 | [分区方案：范围、哈希、一致性哈希与目录](content/zh/hld/trade-offs/13-partitioning-schemes.md) | 四种经典分区策略的决策指南。按查询模式、再平衡成本与热点风险来选。 |
| 14 | [B 树与 LSM 树的存储选型](content/zh/hld/trade-offs/14-btree-vs-lsm.md) | 基于读写比、延迟 SLO 与运维复杂度，在 B 树与 LSM 树存储引擎之间做选择的决策指南。 |
| 15 | [单体与微服务](content/zh/hld/trade-offs/15-monolith-vs-microservices.md) | 为一个镜像你组织架构的架构选择写决策指南。决定因素在于团队规模与领域稳定性，不在流量。 |
| 16 | [复制拓扑：主从、多主与无主](content/zh/hld/trade-offs/16-replication-topologies.md) | 选择写操作如何传播的决策指南：单主、多主、无主，或按 Range 选主的混合方案。 |
| 17 | [分布式事务：2PC vs Saga vs TCC](content/zh/hld/trade-offs/17-distributed-transactions.md) | 按隔离性需求、事务时长和参与者可控程度，在二阶段提交、Saga 补偿与 try-confirm-cancel 三者之间做选择的决策指南。 |
| 18 | [推 vs 拉（扇出、消息传递、信息流）](content/zh/hld/trade-offs/18-push-vs-pull.md) | 数据传播方式这道基本选择题的决策指南：推模式把计算前置到写路径，拉模式把它后置到读路径。最终由扇出比定胜负。 |
| 19 | [Lambda 与 Kappa 架构](content/zh/hld/trade-offs/19-lambda-vs-kappa.md) | 面向分析型数据系统的选型指南：选两条流水线（批处理 + 流处理）还是选一条（只做流处理，靠日志重放补算）。 |
| 20 | [纵向扩展 vs 横向扩展](content/zh/hld/trade-offs/20-vertical-vs-horizontal-scaling.md) | 关于把单机做大还是把机器做多的决策指南。默认路线：先纵向扩展，只有在实测到瓶颈时才横向扩展。 |
| 21 | [范式化与反范式化](content/zh/hld/trade-offs/21-normalization-vs-denormalization.md) | 在严格的 3NF 模式与冗余的预连接数据之间做选择的决策指南，并给出多数生产系统实际采用的 CDC 投影混合模式。 |
| 22 | [单区域与多区域部署](content/zh/hld/trade-offs/22-single-vs-multi-region.md) | 部署地理范围的决策指南：单区域是默认选择，多区域是你能做的最昂贵的选择，而区域固定才是大多数团队真正需要的混合方案。 |

</details>

---

## The DSA Handbook

**数据结构与算法，一个模式一个模式地教。**

[网站](https://dsa.handbook.academy) · [`content/dsa/`](content/dsa/) · [DSA 风格规则](STYLE_GUIDE.md#dsa-specific-deviations)

The DSA Handbook 从 Big-O 和递归讲起，经过线性结构、查找与排序、数组遍历模式，再到树、图、动态规划和进阶字符串算法。最后收在按 API 契约设计数据结构，以及面试本身：认出模式、问对问题、把自己的解法讲清楚。

**一章是怎么搭起来的。** 每章开头是一张速查表，然后深入讲结构或模式，再把 LeetCode 题目当作题卡逐一过：每张卡片带一道例题、一段解法走查和常见错误。Python 解法直接内嵌在正文里。Java、C++、Go 的解法放在章节旁边的 `lc-NNN/` 目录里（`sol.py`、`sol.java`、`sol.cpp`、`sol.go`），各自带测试用例。

### DSA 各部分速览

选一个部分编号直接跳到它的章节，或者展开下面任意一个部分。

| 部分 | 内容简介 | 难度 |
|---|---|---|
| [0](#dsa-part-0) | **基础** —— 大 O、递归、位运算、面试数学、语言惯用法、怎么选语言 | 入门 |
| [1](#dsa-part-1) | **线性数据结构** —— 数组、动态数组内部实现、字符串、哈希表、栈、队列、矩阵 | 入门到进阶 |
| [2](#dsa-part-2) | **查找与排序** —— 线性查找与二分、比较排序与线性时间排序、堆排序、快速选择 | 入门到中级 |
| [3](#dsa-part-3) | **双指针、滑动窗口、前缀和** —— 把 O(n²) 变成 O(n) 的数组遍历模式 | 入门到中级 |
| [4](#dsa-part-4) | **栈与队列模式** —— 单调栈与单调队列、最小/最大栈、表达式解析、用栈实现队列 | 中级 |
| [5](#dsa-part-5) | **链表** —— 指针重连、反转、每 k 个一组反转、Floyd 环检测、合并、LRU 缓存 | 入门到中级 |
| [6](#dsa-part-6) | **树与堆** —— 遍历、Morris、BFS、堆、BST、AVL/红黑树、Trie、树形 DP、线段树 | 入门到进阶 |
| [7](#dsa-part-7) | **递归与回溯** —— 递归模式、通用模板、子集与排列、N 皇后、数独、随机化算法 | 中级到进阶 |
| [8](#dsa-part-8) | **图** —— BFS/DFS、连通分量、拓扑排序、环检测、二分图、并查集、Dijkstra、Bellman-Ford、最小生成树 | 入门到进阶 |
| [9](#dsa-part-9) | **动态规划** —— 记忆化与递推互转、一维/网格/区间/树形/位掩码、背包、LCS、编辑距离、LIS、回文 | 中级到进阶 |
| [10](#dsa-part-10) | **贪心** —— 局部选择何时成立、区间问题、活动选择、霍夫曼编码、跳跃游戏 | 中级 |
| [11](#dsa-part-11) | **位运算** —— 位运算手册、XOR 模式、位掩码技巧、性能技巧 | 中级到进阶 |
| [12](#dsa-part-12) | **字符串与模式匹配** —— 朴素匹配、Rabin-Karp、KMP、Z 数组、Aho-Corasick、后缀数组 | 中级到进阶 |
| [13](#dsa-part-13) | **设计数据结构** —— LRU/LFU、最小栈、计数器、Trie 自动补全、Twitter 时间线、游戏状态 | 中级到进阶 |
| [14](#dsa-part-14) | **面试框架** —— 模式识别、澄清问题、表达、复杂度讨论、模拟面试、按公司的训练路线 | 中级 |

<details>
<summary><a id="dsa-part-0"></a><strong>第 0 部分 · 基础</strong></summary>

**级别：** 入门 &nbsp;·&nbsp; **适合：** 刚开始准备面试、或者停了很久重新捡起来的人。

后面所有内容都默认你已经具备的心智模型和语言手感。Big-O、递归的心智模型、位运算入门、面试真正用得上的数学、Python / Java / C++ / Go 四门语言的习惯写法，以及该选哪门语言去面试。

| # | 章节 | 内容简介 |
|---|---|---|
| 0.0 | [如何使用本手册](content/zh/dsa/part-0-foundations/00-how-to-use.md) | 怎么读这本手册：写给谁、哪些可以跳过、怎么选语言，以及每份 DSA 资料都躲着不答的那四个问题。 |
| 0.1 | [计算复杂度与 Big-O](content/zh/dsa/part-0-foundations/01-complexity-big-o.md) | 计算复杂度、Big-O，以及后面每个算法章节都默认你已掌握的渐进记号词汇。五种 Bachmann-Landau 记号。 |
| 0.2 | [递归的心智模型](content/zh/dsa/part-0-foundations/02-recursion-mental-model.md) | 递归的心智模型：基准情形、递归情形，以及如何在不迷失位置的前提下读懂树状递归。 |
| 0.3 | [位运算入门](content/zh/dsa/part-0-foundations/03-bit-manipulation-primer.md) | 位运算入门：AND、OR、XOR、NOT、移位、popcount。面试题里反复出现的那批位级惯用法。 |
| 0.4 | [面试中的数学](content/zh/dsa/part-0-foundations/04-math-for-interviews.md) | 技术面试里真正用得上的数学：模运算、最大公约数、素性判定、组合数学、整数溢出。 |
| 0.5 | [Python、Java、C++、Go 的语言习惯写法](content/zh/dsa/part-0-foundations/05-language-idioms.md) | Python、Java、C++、Go 的习惯写法：集合、遍历、排序、比较器。每种语言希望你写出的方言。 |
| 0.6 | [选择你的面试语言](content/zh/dsa/part-0-foundations/06-choosing-your-language.md) | 如何为技术面试挑一门语言：权衡、每门语言要你在解题时间上付出什么代价，以及一棵决策树。 |

</details>

<details>
<summary><a id="dsa-part-1"></a><strong>第 1 部分 · 线性数据结构</strong></summary>

**级别：** 入门到进阶 &nbsp;·&nbsp; **适合：** 所有人。这些结构在大多数面试题里都会出现。

连续内存与分桶式的数据结构：数组（静态 / 动态 / 多维）、让 `vector` 成立的摊还 O(1) 倍增规则、把字符串当作带编码的字节数组、哈希表与哈希集合、它们背后的负载因子与冲突计算、栈与队列（配调用栈的类比），以及矩阵操作技巧（原地旋转、螺旋遍历、转置）。

| # | 章节 | 内容简介 |
|---|---|---|
| 1.0 | [数组：静态、动态与多维](content/zh/dsa/part-1-linear-data-structures/00-arrays.md) | 数组：连续内存、O(1) 索引，以及定长数组和动态数组在操作上的差异。 |
| 1.1 | [动态数组内部机制](content/zh/dsa/part-1-linear-data-structures/01-dynamic-array-internals.md) | 动态数组如何增长：均摊 O(1) 追加、翻倍策略、容量与长度的区别，以及各语言里它叫什么。 |
| 1.2 | [字符串：编码、不可变性与构建器](content/zh/dsa/part-1-linear-data-structures/02-strings.md) | 把字符串当作字符数组来看，以及超出这个层次的东西：不可变性、编码、拼接的代价，还有各语言那些能把人问倒的权衡。 |
| 1.3 | [哈希映射与哈希集合](content/zh/dsa/part-1-linear-data-structures/03-hash-maps.md) | 从第一性原理讲哈希映射：哈希函数、桶、平均与最坏复杂度，以及标准库里的实现。 |
| 1.4 | [哈希冲突与装载因子](content/zh/dsa/part-1-linear-data-structures/04-hash-collisions.md) | 哈希冲突：链地址法、开放寻址、装载因子、重哈希，以及哈希表为何在对抗性键下退化成 O(n)。 |
| 1.5 | [栈与调用栈的类比](content/zh/dsa/part-1-linear-data-structures/05-stacks.md) | 栈既是数据结构也是算法基础构件：后进先出（LIFO）、函数调用栈，以及那些一看就该用栈的面试题。 |
| 1.6 | [队列、双端队列与环形缓冲](content/zh/dsa/part-1-linear-data-structures/06-queues-deques.md) | 队列与双端队列：FIFO、双端操作、环形缓冲，以及那些真正该用 deque 的 BFS / 滑动窗口题。 |
| 1.7 | [矩阵操作](content/zh/dsa/part-1-linear-data-structures/07-matrix-manipulation.md) | 二维数组与矩阵操作：原地旋转、转置、螺旋遍历，以及不踩下标越界的索引计算。 |

</details>

<details>
<summary><a id="dsa-part-2"></a><strong>第 2 部分 · 查找与排序</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 每个面试者。光二分查找就会反复出现。

怎么找东西、怎么排序。线性查找，以及它什么时候才是正确答案；不写错边界的标准二分查找；lower_bound / upper_bound / 峰值 / 旋转数组这些变体；比较排序一族（插入、归并、快排，以及 Timsort、Introsort 这类生产级混合实现）；堆排序，以及为什么 `n log n` 是比较排序的下界；线性时间排序（计数、基数、桶）；还有处理 top-k 问题的快速选择。

| # | 章节 | 内容简介 |
|---|---|---|
| 2.0 | [线性查找及其适用场景](content/zh/dsa/part-2-search-sort/00-linear-search.md) | 线性查找：什么时候 O(n) 就是正确答案，以及那些难点在于认出“你根本不需要更聪明的办法”的题目。 |
| 2.1 | [二分查找：标准版本](content/zh/dsa/part-2-search-sort/01-binary-search-canonical.md) | 把二分查找写对：标准模板、差一错误陷阱，以及动笔写循环之前必须先讲清楚的不变量。 |
| 2.2 | [二分查找的变体：lower_bound、upper_bound、峰值与旋转数组](content/zh/dsa/part-2-search-sort/02-binary-search-variants.md) | 超越有序数组的二分查找：lower_bound、upper_bound、first-true、last-true，以及在答案空间上二分。 |
| 2.3 | [比较排序 I：插入排序与归并排序](content/zh/dsa/part-2-search-sort/03-comparison-sorts-1.md) | 插入排序、选择排序与冒泡排序：O(n^2) 这一档排序，各自在真实输入上何时占优，以及藏在它们内部的常数因子教训。 |
| 2.4 | [比较排序 II：快速排序、分区与生产级混合排序](content/zh/dsa/part-2-search-sort/04-comparison-sorts-2.md) | 归并排序与快速排序：分治、分区、稳定性，以及为什么生产库里的排序都是混合算法。 |
| 2.5 | [堆排序与 n log n 下界](content/zh/dsa/part-2-search-sort/05-heap-sort.md) | 堆排序与二叉堆：堆化、上浮、下沉，以及为什么堆是 k 路归并与优先队列的正确工具。 |
| 2.6 | [线性时间排序：计数排序、基数排序、桶排序](content/zh/dsa/part-2-search-sort/06-linear-sorts.md) | 计数排序、基数排序、桶排序：当输入具备结构时，这三类 O(n) 排序能突破比较排序的下界。 |
| 2.7 | [快速选择：线性时间选择](content/zh/dsa/part-2-search-sort/07-quickselect.md) | 快速选择：O(n) 期望的第 k 小、分区过程，以及那些只需选出所需的那一部分、从而绕开完整排序的题目。 |

</details>

<details>
<summary><a id="dsa-part-3"></a><strong>第 3 部分 · 双指针、滑动窗口、前缀和</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 那些嵌套循环解法总是超时的人。

三个关系很近的模式，说到底都是“更聪明地走数组”：相向双指针（Container With Most Water、3Sum）、同向双指针、定长与变长滑动窗口、前缀和与差分数组，以及用前缀和加哈希表处理“子数组和等于 K”这一类问题。

| # | 章节 | 内容简介 |
|---|---|---|
| 3.0 | [双指针：两端相向](content/zh/dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md) | 双指针之两端相向：向内收缩的扫掠、有序数组上的不变量，以及经典的「和为定值」与「约束下的数对」模式。 |
| 3.1 | [双指针：同向](content/zh/dsa/part-3-pointers-window-prefix/01-two-pointers-same-direction.md) | 双指针同向：快慢指针、原地过滤，以及那些把这套模式伪装起来的数组题。 |
| 3.2 | [滑动窗口：固定长度](content/zh/dsa/part-3-pointers-window-prefix/02-sliding-window-fixed.md) | 固定长度的滑动窗口：每步用 O(1) 维护一个滑动聚合值，以及那些约束就是窗口长度的题目。 |
| 3.3 | [滑动窗口：变长窗口](content/zh/dsa/part-3-pointers-window-prefix/03-sliding-window-variable.md) | 变长滑动窗口：不变量被破坏时如何收缩，以及最长子串 / 最小窗口这一族问题。 |
| 3.4 | [前缀和与差分数组](content/zh/dsa/part-3-pointers-window-prefix/04-prefix-sums.md) | 前缀和：O(1) 的区间求和查询、二维扩展，以及在预处理到位后一下子变简单的子数组求和问题。 |
| 3.5 | [前缀和 + 哈希表组合](content/zh/dsa/part-3-pointers-window-prefix/05-prefix-sum-hash.md) | 前缀和 + 哈希表：数出和等于目标值的子数组，经典的子数组和等于 K 题族，以及取模一类的推广。 |

</details>

<details>
<summary><a id="dsa-part-4"></a><strong>第 4 部分 · 栈与队列模式</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 卡在下一个更大元素和滑动窗口最大值这两类题上的人。

把栈和队列当作*算法模式*，而不只是数据结构。单调栈（Daily Temperatures、Largest Rectangle in Histogram）、单调双端队列（Sliding Window Maximum）、最小/最大栈（push/pop 之下 O(1) 查最小值）、表达式解析（调度场算法 / 逆波兰表达式），以及用栈实现队列的摊还论证。

| # | 章节 | 内容简介 |
|---|---|---|
| 4.0 | [单调栈](content/zh/dsa/part-4-stack-queue-patterns/00-monotonic-stack.md) | 压入的是下标，来值一旦违反单调顺序就弹出。这个数据结构专为两件事的交集而生：最近更大值查询，以及一趟线性扫描。 |
| 4.1 | [单调双端队列](content/zh/dsa/part-4-stack-queue-patterns/01-monotonic-deque.md) | 两端弹出，维护带窗口的滚动最大值或最小值，每次滑动摊还 O(1)。它是让滑动窗口极值类问题从 n log k 变成线性的那个结构。 |
| 4.2 | [最小栈与最大栈](content/zh/dsa/part-4-stack-queue-patterns/02-min-max-stack.md) | 在栈旁边钉一个平行的辅助栈，让聚合查询做到 O(1)；再把同一套手法从动态最小值推广到延迟批量更新，最后到频率跟踪。 |
| 4.3 | [表达式求值与解析](content/zh/dsa/part-4-stack-queue-patterns/03-expression-evaluation.md) | 栈把运算符挂起，直到它的右操作数出现。调度场算法的节奏，以及 LeetCode 224/227/772 这一族题：字符到达的顺序，并不等于算术应当发生的顺序。 |
| 4.4 | [用栈实现队列](content/zh/dsa/part-4-stack-queue-patterns/04-queue-from-stacks.md) | 只用栈的原语实现一个队列。双栈惰性转移的技巧、每个操作的均摊 O(1)，以及为什么从 FIFO 反推 LIFO 这个逆问题没有对称构造。 |

</details>

<details>
<summary><a id="dsa-part-5"></a><strong>第 5 部分 · 链表</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 在白板上画方框加箭头、画着画着就迷路的人。

链表就是指针外科手术。哨兵节点模式、标准的迭代反转（以及它的递归写法）、每 k 个一组反转、Floyd 龟兔赛跑判环（以及环起点那个公式为什么成立）、合并有序链表，还有作为哈希表加双向链表经典组合的 LRU 缓存。

| # | 章节 | 内容简介 |
|---|---|---|
| 5.0 | [链表基础：哨兵、指针重连与双向链表设计](content/zh/dsa/part-5-linked-lists/00-linked-list-fundamentals.md) | 哨兵节点与哑节点抹掉了为头尾单开分支的边界判断。指针重连这套纪律，以及第 5 部分后面每一章都默认你已经掌握的双向链表设计。 |
| 5.1 | [反转模式](content/zh/dsa/part-5-linked-lists/01-reversal-patterns.md) | 用 prev、curr、next 三个指针原地反转单链表。先快照、再覆盖的规则，以及迭代与递归两种写法——它们会作为基本操作出现在后面每一篇链表章节里。 |
| 5.2 | [每 k 个一组反转](content/zh/dsa/part-5-linked-lists/02-reverse-in-groups-k.md) | 三个机械阶段叠加在单链表反转之上，就能用 O(1) 额外内存解掉 LeetCode 25，并且干净地处理组与组之间的接缝。 |
| 5.3 | [环检测（Floyd 龟兔算法）](content/zh/dsa/part-5-linked-lists/03-cycle-detection.md) | 两个指针，一个的步速是另一个的两倍，用 O(n) 时间、O(1) 空间定位环的入口节点；再用一段三行的模运算证明说明第二阶段的重置为什么成立。 |
| 5.4 | [合并链表](content/zh/dsa/part-5-linked-lists/04-merging-linked-lists.md) | 原地合并有序链表。哑头节点如何消掉首节点的特例判断，双链表与 k 链表两个版本，以及 LC 23 的堆解与分治解。 |
| 5.5 | [LRU 缓存：哈希映射 + 双向链表](content/zh/dsa/part-5-linked-lists/05-lru-cache.md) | 用一条双向链表加一张哈希映射，搭出 LeetCode 146 背后的数据结构。`get` 和 `put` 都是 O(1)，链表操作全部手写，不走 Python OrderedDict 的捷径。 |

</details>

<details>
<summary><a id="dsa-part-6"></a><strong>第 6 部分 · 树与堆</strong></summary>

**级别：** 入门到进阶 &nbsp;·&nbsp; **适合：** 已经过了准备阶段线性数据结构那一关的人。

层次结构与优先队列。二叉树基础、三种深度优先遍历（前/中/后序，递归和迭代都写）、O(1) 空间的 Morris 遍历、层序 BFS、堆与优先队列（O(n) 建堆）、二叉搜索树、AVL 旋转、红黑树概览、Trie、树形 DP 入门（后序加附带状态），以及区间查询用的线段树简介。

| # | 章节 | 内容简介 |
|---|---|---|
| 6.0 | [二叉树基础](content/zh/dsa/part-6-trees-heaps/00-binary-tree-fundamentals.md) | 树的术语、高度与深度的区别、两种物理表示，以及第 6 部分后面每一章都默认你已经钉死的那套词汇。 |
| 6.1 | [树的遍历：前序、中序、后序](content/zh/dsa/part-6-trees-heaps/01-tree-traversals.md) | 同一套递归骨架，visit() 只有三个可放的位置。前序、中序、后序三种放法，在同一副脚手架上解决不同的题目家族。 |
| 6.2 | [Morris 遍历：用线索实现 O(1) 空间的中序遍历](content/zh/dsa/part-6-trees-heaps/02-morris-traversal.md) | 不借助递归栈，也不占额外内存，靠从每棵子树的右端后代临时连回其后继的右指针，完成二叉树的中序遍历。 |
| 6.3 | [层序遍历：树上的 BFS](content/zh/dsa/part-6-trees-heaps/03-level-order-traversal.md) | 树上的 BFS，关键在于 level_size = len(queue) 这个快照——它把「访问每个节点」变成「按深度分组、按顺序访问每个节点」，并一路保住层与层之间的边界。 |
| 6.4 | [堆与优先队列](content/zh/dsa/part-6-trees-heaps/04-heaps-priority-queues.md) | 上滤与下滤、Floyd 的 O(n) 自底向上建堆与 O(n log n) 反复插入的对比，以及优先队列这一原语在各语言里的 API 差异。 |
| 6.5 | [二叉搜索树](content/zh/dsa/part-6-trees-heaps/05-binary-search-trees.md) | BST 不变式、查找/插入/删除三种操作，以及为什么每个生产可用的标准库都用平衡变体而不是朴素 BST。 |
| 6.6 | [AVL 树与旋转](content/zh/dsa/part-6-trees-heaps/06-avl-rotations.md) | 从第一性原理讲自平衡二叉搜索树：AVL 不变式、四种旋转情形、1.44 log(n+2) 的高度上界，以及为什么生产环境里的每个有序映射都可以追溯到 1962 年那篇论文。 |
| 6.7 | [红黑树：概览](content/zh/dsa/part-6-trees-heaps/07-red-black-overview.md) | TreeMap、std::map 和 Linux 调度器背后的平衡方案。五条不变量、高层视角下的旋转情形，以及一场面试可以合理地追问到什么程度。 |
| 6.8 | [Trie](content/zh/dsa/part-6-trees-heaps/08-tries.md) | 子节点映射加终止标记的节点，让前缀遍历的时间只取决于查询长度，为 O(L)。自动补全与 IP 路由这一族问题里，前缀结构就是全部要害。 |
| 6.9 | [树形 DP 入门：后序遍历与旁路状态](content/zh/dsa/part-6-trees-heaps/09-tree-dp.md) | 后序辅助函数把旁路状态通过参数往下传，把聚合值沿调用栈往上返回，再用一个全局变量记录直径式“穿过本节点”的最优值。 |
| 6.10 | [线段树](content/zh/dsa/part-6-trees-heaps/10-segment-tree.md) | 在同一份数组上做到 O(log n) 单点更新与 O(log n) 区间查询。递归构建，以及为支持区间更新而加的懒标记下传扩展。 |

</details>

<details>
<summary><a id="dsa-part-7"></a><strong>第 7 部分 · 递归与回溯</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 子集、排列、组合的解法全靠猜的人。

回溯不过是带上状态恢复纪律的 DFS。递归的几种形态（线性、树形、分治）；一套能套到任何约束问题上的回溯模板；子集、组合与排列；带剪枝的 N 皇后；用约束传播和前向检查解数独；网格上的单词搜索；以及随机算法用的 Fisher-Yates 洗牌、蓄水池抽样和拒绝采样。

| # | 章节 | 内容简介 |
|---|---|---|
| 7.0 | [递归模式：线性、树形与分治](content/zh/dsa/part-7-recursion-backtracking/00-recursion-patterns.md) | 三种调用树形状——线性、树形、分治——以及如何从递归分支读出形状，从而判断自己到底需要记忆化、累加器，还是两者都不用。 |
| 7.1 | [回溯模板](content/zh/dsa/part-7-recursion-backtracking/01-backtracking-template.md) | 选择、递归、撤销，而“依据部分信息剪枝”正是把指数级搜索变成能跑完的那一步。 |
| 7.2 | [子集、组合、排列](content/zh/dsa/part-7-recursion-backtracking/02-subsets-combinations-permutations.md) | 同一个递归配三个旋钮（何时快照 path、跳什么、何时允许重复），覆盖子集（LC 78）、排列（LC 46）与组合总和（LC 39）。 |
| 7.3 | [N-Queens：剪枝与约束传播](content/zh/dsa/part-7-recursion-backtracking/03-n-queens.md) | 列与对角线的占用位集让攻击判定降到 O(1)。对称性剪枝，以及那套能推广到数独之外的约束传播思路。 |
| 7.4 | [数独求解：约束传播与前向检查](content/zh/dsa/part-7-recursion-backtracking/04-sudoku-solver.md) | 把行、列、宫三组 AllDiff 约束编码成位掩码，配上 MRV 选格和一轮前向检查循环，即便候选填法多达 4 × 10^38 种，9x9 数独也能在毫秒内解完。 |
| 7.5 | [单词搜索与网格回溯](content/zh/dsa/part-7-recursion-backtracking/05-word-search.md) | 带标记-恢复的网格 DFS：用原地访问标记，把一道二维回溯题化成一次干净的递归。 |
| 7.6 | [随机算法：Fisher-Yates、蓄水池抽样与拒绝采样](content/zh/dsa/part-7-recursion-backtracking/06-randomized-algorithms.md) | 三份随机化配方——Fisher-Yates 洗牌、蓄水池抽样、拒绝采样——真正要学的是它们均匀性的正确性证明，而不是代码。 |

</details>

<details>
<summary><a id="dsa-part-8"></a><strong>第 8 部分 · 图</strong></summary>

**级别：** 入门到进阶 &nbsp;·&nbsp; **适合：** 准备 FAANG 级别面试的人——图论题就是分水岭。

图这一组章节从头到尾都是面试级。邻接表与邻接矩阵两种表示；作为两种正交遍历模板的 BFS 和 DFS；连通分量与泛洪填充（Number of Islands）；用 Kahn 的队列实现拓扑排序，以及用 DFS 后序反转实现；有向图和无向图上的判环；二分图判定；带路径压缩和按秩合并的并查集；Dijkstra 算法；Bellman-Ford 与负环；还有 Kruskal 和 Prim 两种最小生成树。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.0 | [图的表示](content/zh/dsa/part-8-graphs/00-graph-representation.md) | 存储图的三种方式（邻接表、邻接矩阵、边列表），决定取舍的内存账，以及每场面试都默认的那个选择。 |
| 8.1 | [广度优先搜索](content/zh/dsa/part-8-graphs/01-bfs.md) | 队列、已访问集合，以及同一个顶点绝不入队两次的纪律。无权重边上的最短路径算法，外加它的多源与网格变体。 |
| 8.2 | [深度优先搜索](content/zh/dsa/part-8-graphs/02-dfs.md) | 五行代码：进入时访问、在邻居上递归；外加几个定制旋钮（进入钩子、退出钩子、边分类）——Part 8 后面每一个高级图算法都是在拧它们。 |
| 8.3 | [连通分量与洪水填充](content/zh/dsa/part-8-graphs/03-connected-components.md) | 统计并度量图或网格中的连通分量：那层外层扫描把普通的 BFS 或 DFS 变成了算法。 |
| 8.4 | [拓扑排序：Kahn 算法](content/zh/dsa/part-8-graphs/04-topological-sort-kahn.md) | 按入度驱动、基于队列、迭代求解。Kahn 算法靠无法消费全部顶点来标记环的存在。 |
| 8.5 | [拓扑排序：DFS 后序反转](content/zh/dsa/part-8-graphs/05-topological-sort-dfs.md) | 基于 DFS 的拓扑排序：三色递归、后序输出，以及从同一副骨架里长出来的算法——Tarjan 桥、Hierholzer 欧拉路径、环长检测。 |
| 8.6 | [图中的环检测](content/zh/dsa/part-8-graphs/06-cycle-detection-graphs.md) | 有向图用三状态 DFS 染色，无向图用带 parent 参数的单状态 DFS；以及用错其中一种时，两半共有的那个 bug。 |
| 8.7 | [二分图判定](content/zh/dsa/part-8-graphs/07-bipartite-check.md) | 用 BFS 或 DFS 给图二染色。奇环这个障碍、线性时间的判定方法，以及把这个问题伪装成“分成两组”的那一族题目。 |
| 8.8 | [Union-Find：父指针森林、路径压缩与按秩合并](content/zh/dsa/part-8-graphs/08-union-find.md) | 父指针森林、按秩合并、路径压缩，以及反阿克曼摊还界。它还包办了环检测与连通分量这一整族问题。 |
| 8.9 | [Dijkstra 最短路径算法](content/zh/dsa/part-8-graphs/09-dijkstra.md) | 非负权边加权图上的单源最短路径：优先队列的运作机制、惰性删除惯用法，以及最小-最大路径与 0-1 BFS 两种特化形式。 |
| 8.10 | [Bellman-Ford 与负环](content/zh/dsa/part-8-graphs/10-bellman-ford.md) | 每条边都松弛 V-1 轮，再跑第 V 趟检测负环，以及那个专门收拾 Dijkstra 会悄悄报错的负权边情形的算法。 |
| 8.11 | [最小生成树：Kruskal 算法](content/zh/dsa/part-8-graphs/11-mst-kruskal.md) | Kruskal 算法：按权重排好边，逐条接受两端点分属不同连通分量的边，凑够 n-1 条就停。环检测交给并查集，代价全在排序上。 |
| 8.12 | [最小生成树：Prim 算法](content/zh/dsa/part-8-graphs/12-mst-prim.md) | Prim 算法从一个起始顶点出发，不断吸收横跨树边界的最便宜边，把最小生成树长出来。和 Kruskal 得到同一棵 MST，走的是另一条路；在稠密图和隐式图上更优。 |

</details>

<details>
<summary><a id="dsa-part-9"></a><strong>第 9 部分 · 动态规划</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 大多数候选人最怕的一部分。如果你觉得“DP 题就是不开窍”，读这里。

全书最吃力的一部分。从递归（自顶向下记忆化）把 DP 搭起来，再重新推导出自底向上的表格法；一维状态的 DP（Climbing Stairs、House Robber、决策型 DP）；字符串前缀决策型 DP（Decode Ways、Word Break）；0/1 背包与完全背包；最长公共子序列与编辑距离；O(n²) 的 LIS 与耐心排序的 O(n log n) 版本；回文 DP；区间 DP（矩阵链、戳气球）；网格 DP；树形 DP；以及处理“所有子集”类问题的状压 DP 家族。

| # | 章节 | 内容简介 |
|---|---|---|
| 9.0 | [动态规划：从递归到记忆化](content/zh/dsa/part-9-dynamic-programming/00-dp-recursion-to-memo.md) | 当递归的调用树带着冗余时，八个字符的缓存就能把斐波那契从 O(phi^n) 降到 O(n)。以及从朴素递归到记忆化递归、再到自底向上打表的系统化动作。 |
| 9.1 | [DP：自底向上制表](content/zh/dsa/part-9-dynamic-programming/01-dp-bottom-up-tabulation.md) | 把记忆化递归改写成迭代填表：依赖 DAG、拓扑序、循环、数组。外加滚动对的 O(1) 空间压缩。 |
| 9.2 | [一维状态上的动态规划](content/zh/dsa/part-9-dynamic-programming/02-dp-1d.md) | 一维状态 DP：位置 i 的答案只依赖前面一个很小的固定窗口。爬楼梯、打家劫舍、使用最小花费爬楼梯，以及用滚动标量把空间压掉。 |
| 9.3 | [Decode Ways 与 Word Break：字符串前缀决策 DP](content/zh/dsa/part-9-dynamic-programming/03-dp-decode-word-break.md) | 两道字符串前缀决策 DP：递推式长得不一样，跑的却是同一套 `dp[i]` = OR-over-valid-splits 骨架。LC 91 Decode Ways 与 LC 139 Word Break。 |
| 9.4 | [0/1 背包](content/zh/dsa/part-9-dynamic-programming/04-knapsack-01.md) | 0/1 背包：每个物品取或不取、一维表格的填法、内层循环为何从右往左扫，以及一族子集和重述（LC 416、494、474、879、1049）。 |
| 9.5 | [无界背包：物品可以被反复选择](content/zh/dsa/part-9-dynamic-programming/05-knapsack-unbounded.md) | 相对 0/1 背包只把内层循环的方向反过来，就解锁了零钱兑换这一族问题：最少硬币数、方案数、最小成本如何共用同一条递推式。 |
| 9.6 | [最长公共子序列](content/zh/dsa/part-9-dynamic-programming/06-lcs.md) | 用 O(n*m) 的二维 DP 求最长公共子序列，用父指针回溯还原出真正的子序列，以及共用同一张表的编辑距离与最短公共超序列变体。 |
| 9.7 | [编辑距离](content/zh/dsa/part-9-dynamic-programming/07-edit-distance.md) | Wagner-Fischer 的三路递推式：求把一个字符串变成另一个所需的最少插入、删除、替换次数。二维字符串 DP 的典范。 |
| 9.8 | [最长递增子序列：平方级 DP](content/zh/dsa/part-9-dynamic-programming/08-lis-quadratic.md) | 按每个下标定义 `dp[i]` = 在更小的前驱里取最大，就得到 O(n^2) 的 LIS。还能用父指针还原出一条子序列，并推广到若干个仍属平方级的变体（LIS 计数、带 k 容差的 LIS）。 |
| 9.9 | [最长递增子序列：耐心排序](content/zh/dsa/part-9-dynamic-programming/09-lis-patience-sort.md) | 从 9.8 的 O(n^2) DP 走到 O(n log n)：耐心排序的视角、tails 数组的不变式、按尾值二分查找，以及 LC 354 先排序再求 LIS 的归约。 |
| 9.10 | [回文 DP](content/zh/dsa/part-9-dynamic-programming/10-palindrome-dp.md) | 递推式要读 `dp[i+1][j-1]` 的二维表，逼着你按对角线方向填。最长回文子串与最少切割这一族题，都长在这张表上。 |
| 9.11 | [区间 DP：矩阵链与戳气球](content/zh/dsa/part-9-dynamic-programming/11-interval-dp.md) | 状态是 `dp[l][r]`，递推式挑选区间内最后执行的那步操作。戳气球（LC 312）与矩阵链乘法（LC 1039）是两道标准范例。 |
| 9.12 | [网格 DP：正向填表，反向求生](content/zh/dsa/part-9-dynamic-programming/12-grid-dp.md) | 自顶向下的正向填表（不同路径、最小路径和），与自底向上的反向求生（地牢游戏）。方向由约束落在起点还是终点决定。 |
| 9.13 | [树形 DP：沿调用栈向上传递的状态](content/zh/dsa/part-9-dynamic-programming/13-tree-dp.md) | 后序遍历的辅助函数把不止一个数字交还给父节点 —— (rob, skip)、(diameter, depth) —— 围绕 LC 337 House Robber III 与 LC 124 Max Path Sum 展开。 |
| 9.14 | [位掩码 DP](content/zh/dsa/part-9-dynamic-programming/14-bitmask-dp.md) | 把一个小规模全集中的一个子集编码成一个整数，`dp[mask]` = ...，以及指派 / TSP / 走遍所有节点的最短路这一族问题——只有这个状态放得下。 |

</details>

<details>
<summary><a id="dsa-part-10"></a><strong>第 10 部分 · 贪心</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 被“看起来没错”的贪心坑出过 Wrong Answer 的人。

局部选择什么时候能给出全局最优，以及怎么证明。把贪心思路放在 DP 的对照下讲；区间调度（排序比较器*就是*算法本身）；活动选择与任务调度器那一族；Huffman 编码；以及作为“扫一遍、维护一个运行中的不变量”这类标准题的 Jump Games / Gas Station。

| # | 章节 | 内容简介 |
|---|---|---|
| 10.0 | [贪心思维：局部最优何时成立，何时不成立](content/zh/dsa/part-10-greedy/00-greedy-thinking.md) | 当局部最优的选择可以被证明能叠加出全局最优答案时。交换论证这一证明技巧，以及贪心看似正确、实则不成立的失效模式。 |
| 10.1 | [区间调度：比较器就是算法](content/zh/dsa/part-10-greedy/01-interval-scheduling.md) | 按结束时间排序，你就选出了最大的不重叠子集。整个算法就是那个比较器——不是 DP，不是堆。 |
| 10.2 | [活动选择与任务调度器家族](content/zh/dsa/part-10-greedy/02-activity-selection.md) | 剩余任务的大顶堆配上冷却队列。同一套骨架能解 LC 621 Task Scheduler、会议室预订与会议排期。 |
| 10.3 | [霍夫曼编码](content/zh/dsa/part-10-greedy/03-huffman-encoding.md) | 反复合并两个频率最低的符号，自底向上构造出最优前缀码。这个构造的最优性证明，以及它为什么能赢下自顶向下构造失败的地方。 |
| 10.4 | [跳跃游戏与加油站](content/zh/dsa/part-10-greedy/04-jump-games-gas-station.md) | 两道看起来要 O(n^2) 的数组题，都靠一趟线性扫描维护一个滚动量解决。LC 55 Jump Game 与 LC 134 Gas Station。 |

</details>

<details>
<summary><a id="dsa-part-11"></a><strong>第 11 部分 · 位运算</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 准备偏底层、偏性能的面试的人（HFT、嵌入式、内核、GPU）。

位运算手册（置位/清位/翻转、最低置位、popcount）；XOR 模式（Single Number I/II/III、Missing Number）；把位掩码当作紧凑的子集状态；以及应对“你代码是对的，把它再快 10 倍”这类面试轮的位级性能技巧。

| # | 章节 | 内容简介 |
|---|---|---|
| 11.0 | [位运算速查手册](content/zh/dsa/part-11-bit-manipulation/00-bit-operations-cookbook.md) | 第 11 部分其余内容所依赖的两条恒等式：x & -x 取出最低置位位，x & (x-1) 把它消掉。另外还有 popcount、置位/清零/翻转/测试，以及子集遍历。 |
| 11.1 | [XOR 模式](content/zh/dsa/part-11-bit-manipulation/01-xor-patterns.md) | XOR 模式：把配对抵消看成位向量上的群运算，以及它用 O(n) 时间、O(1) 空间解决的那一族 LC 题目。 |
| 11.2 | [位掩码技巧](content/zh/dsa/part-11-bit-manipulation/02-bitmask-techniques.md) | 把小集合编码成一个整数；遍历每个子集；走遍某个掩码的所有子掩码。位掩码 DP 背后的底座。 |
| 11.3 | [高性能位运算技巧](content/zh/dsa/part-11-bit-manipulation/03-bit-tricks-performance.md) | 一套位模式恒等式工具箱——Brian Kernighan 循环、x & -x、SWAR popcount、de Bruijn 位扫描——以及那些让其中大部分在生产环境里已经过时的硬件内建指令。 |

</details>

<details>
<summary><a id="dsa-part-12"></a><strong>第 12 部分 · 字符串与模式匹配</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 面试岗位把字符串当重点的人（搜索基础设施、NLP 基础设施、IDE、编译器工具链）。

超出朴素 O(n·m) 基线的子串搜索。带滚动哈希的 Rabin-Karp；KMP 与失配函数（以及它为什么是前缀函数的一般化）；Z 算法；一趟扫描就能匹配多个模式的 Aho-Corasick；以及面向“所有子串”类查询的后缀数组简介。

| # | 章节 | 内容简介 |
|---|---|---|
| 12.0 | [朴素字符串匹配](content/zh/dsa/part-12-strings-pattern-matching/00-naive-string-matching.md) | 藏在 str.find 与 str.indexOf 背后的 O(nm) 双重循环。它在真实文本上的平均表现，以及那些逼出 KMP 与 Rabin-Karp 的对抗性输入。 |
| 12.1 | [Rabin-Karp 与滚动哈希](content/zh/dsa/part-12-strings-pattern-matching/01-rabin-karp.md) | 滚动哈希让固定长度窗口的哈希维护降到每次滑动 O(1)。双哈希击破对抗性碰撞，也顺手打开了重复子串这一族问题。 |
| 12.2 | [KMP 与失败函数](content/zh/dsa/part-12-strings-pattern-matching/02-kmp.md) | Knuth-Morris-Pratt：用失败函数做到确定性的 O(n+m) 子串查找，以及那张单靠自己就能解决一小类问题的前缀-后缀表。 |
| 12.3 | [Z 算法](content/zh/dsa/part-12-strings-pattern-matching/03-z-algorithm.md) | 把 Z 数组理解成字符串上的前缀匹配表；让算法保持线性的 [l, r) 窗口不变量；以及它与 KMP 在精确模式匹配上的对比。 |
| 12.4 | [Aho-Corasick：多模式，一趟扫完](content/zh/dsa/part-12-strings-pattern-matching/04-aho-corasick.md) | 带失败链接的 Trie：Aho-Corasick 如何只扫一遍文本，就在 O(n + m + z) 时间内报出固定词典中每个模式的每一次出现。 |
| 12.5 | [后缀数组：把所有后缀排好序的索引](content/zh/dsa/part-12-strings-pattern-matching/05-suffix-array.md) | 用大约 4n 字节存下所有后缀的有序索引，再加一个 LCP 数组。基于这两个数组做子串搜索、最长重复子串与 BWT 应用。 |

</details>

<details>
<summary><a id="dsa-part-13"></a><strong>第 13 部分 · 设计数据结构</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** SDE2 及以上的候选人——“设计 X”是高级编码轮的常备题型。

DSA 与 HLD 之间的桥：怎么把基础构件组合成满足“每个操作一条复杂度”契约的 API。完整的 LRU 讲法，含并发与多级部署框架；用按频率分桶的双向链表实现 LFU；最小栈与最大频率栈；命中计数器与限流器；Trie 支撑的自动补全；Twitter 信息流设计；还有一段游戏状态设计的小案例。

| # | 章节 | 内容简介 |
|---|---|---|
| 13.0 | [LRU 缓存：设计框架、并发与多级部署](content/zh/dsa/part-13-design-the-data-structure/00-lru-cache.md) | 指针重连只是 LeetCode 146 的热身。线程安全、多级部署，以及 Caffeine 那类准入策略，才是真正的面试。 |
| 13.1 | [LFU 缓存：哈希表加频率分桶的双向链表](content/zh/dsa/part-13-design-the-data-structure/01-lfu-cache.md) | 一张以频率为键的哈希表，每个值是一串频率相同的条目组成的双向链表，再加一个最小频率跟踪器。正是这个构造让 LeetCode 460 拿到了 Hard 难度评级。 |
| 13.2 | [最小栈与最大频率栈](content/zh/dsa/part-13-design-the-data-structure/02-min-stack-maxfreq.md) | 两个设计题，一套技法。给栈配一个平行的辅助结构，被增强的操作就塌缩成一次 peek；辅助结构的形状跟着契约走。 |
| 13.3 | [命中计数器与限流器](content/zh/dsa/part-13-design-the-data-structure/03-hit-counter-rate-limiter.md) | LC 362 Design Hit Counter 的固定窗口、滑动日志窗口与令牌桶等几种变体，以及生产系统（Cloudflare、Stripe、AWS）在精度与内存之间做的取舍。 |
| 13.4 | [Trie 自动补全](content/zh/dsa/part-13-design-the-data-structure/04-trie-autocomplete.md) | 每节点带排序的 top-k 缓存、跨按键持续的补全建议，以及把一个朴素 trie 从 O(L + 答案数) 推到个位数微秒 typeahead 的那些设计动作。 |
| 13.5 | [Twitter 信息流设计](content/zh/dsa/part-13-design-the-data-structure/05-twitter-feed.md) | 以 LC 355 Design Twitter 为例，讲解一个由机制驱动的设计类：单调时间戳、按用户划分的只追加日志，以及通过作者游标堆为信息流做 k 路归并。 |
| 13.6 | [游戏状态设计](content/zh/dsa/part-13-design-the-data-structure/06-game-state-design.md) | 存棋盘会给出的那个答案，而不是棋盘本身。按行、按列、按对角线的计数器让 LC 348 Tic-Tac-Toe 每次落子 O(1)、总内存 O(n)。 |

</details>

<details>
<summary><a id="dsa-part-14"></a><strong>第 14 部分 · 面试框架</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 距离真实面试窗口不到一个月的人。

把“解法正确”变成“这一轮通过”的元技能。模式识别训练；开头五分钟该问什么问题的脚本；一边写代码一边怎么讲；怎么把复杂度讨论得让人信服；怎么把模拟面试跑出效果；最常见的坑；Amazon 领导力准则（简版与口述版）；Meta 的 AI 辅助轮（形式、提示技巧、常见失误）；以及 per-company tracks 索引，指向各家公司的专属阅读清单。

| # | 章节 | 内容简介 |
|---|---|---|
| 14.0 | [模式识别：见效最快的那项技能](content/zh/dsa/part-14-interview-framework/00-pattern-recognition.md) | 覆盖应届生 FAANG 编码轮约 95% 的二十个模式，其中前五个自己就占一半，以及在头 90 秒里叫出模式名的识别节奏。 |
| 14.1 | [头五分钟：澄清问题](content/zh/dsa/part-14-interview-framework/01-clarifying-questions.md) | 头五分钟该问什么。面试官真正在打分的那些问题、能体现资深度的提问方式，以及让最多候选人输掉整轮的那一种失败模式。 |
| 14.2 | [面试过程中的沟通](content/zh/dsa/part-14-interview-framework/02-communicating.md) | 出声思考、把权衡讲出来，以及 interviewing.io 那组数据：代码 4 分、解题 4 分、沟通 2 分的候选人，结果反而好过 3-3-4 的那一位。 |
| 14.3 | [复杂度的讨论](content/zh/dsa/part-14-interview-framework/03-complexity-discussions.md) | 如何用一句话回答“时间复杂度是多少”：推导而不是背答案、时间与空间一起说、修饰词选对（最坏/平均/摊还），四个信号一次全给。 |
| 14.4 | [模拟面试流程](content/zh/dsa/part-14-interview-framework/04-mock-interview-process.md) | 什么时候开始打模拟面试、该用哪个平台、真正需要多少场，以及如何做复盘才能把一场会话转化成信号。 |
| 14.5 | [常见陷阱](content/zh/dsa/part-14-interview-framework/05-common-pitfalls.md) | FAANG 级面试中最具杀伤力的五种失败模式：直接开写、闷头解题、漏掉澄清、复杂度答得没结构，以及那些看起来像能力不足、实则是信号缺失的失误。 |
| 14.6 | [亚马逊领导力准则速览](content/zh/dsa/part-14-interview-framework/06-amazon-lps-brief.md) | 哪些领导力准则在编程轮里出现得最多，为什么第 5 分钟和第 50 分钟的闲聊也算在评分表里，以及哪些轮次里“我只要把算法做得出色”会翻车。 |
| 14.7 | [Amazon 领导力准则的讲述](content/zh/dsa/part-14-interview-framework/07-amazon-lps-narration.md) | 面向技术轮的 STAR 格式微型叙述，以及行为轮和 Bar Raiser 轮期待候选人带着进场的八到十二个完整故事。 |
| 14.8 | [Meta 的 AI 轮：具体形式](content/zh/dsa/part-14-interview-framework/08-meta-ai-round-format.md) | 2026 年 Meta 的 AI 辅助编码轮到底是什么：60 分钟、三个阶段、一份模型菜单，以及一套更看重你怎么用 AI、而不是 AI 产出什么的评分标准。 |
| 14.9 | [Meta 的 AI 面试轮：提示词策略](content/zh/dsa/part-14-interview-framework/09-meta-ai-round-prompting.md) | AI 辅助编码轮里的提示词策略。三层级——指挥者、抽检者、打字员——产出的行为，正是评分标准要打的那套行为。 |
| 14.10 | [Meta 的 AI 轮：常见失败](content/zh/dsa/part-14-interview-framework/10-meta-ai-round-failures.md) | 把助手当成解题器的候选人、讲不清自己代码的候选人，以及那套让“AI 会抓出 bug”这类准备彻底失效的系统提示词加固。 |
| 14.11 | [按公司划分的线路：索引与用法](content/zh/dsa/part-14-interview-framework/11-per-company-tracks-index.md) | 什么时候该打开一份按公司划分的线路，怎么把它叠在第 0 至 13 部分之上，以及三个信号告诉你该从模式练习转向面向具体公司的校准。 |

</details>

### 模式决策页

一道题有两种解法时（递归还是迭代、BFS 还是 DFS、滑动窗口还是前缀和），决策页把两者摆在一起比。每一页都有一棵决策树、一张“选 A 还是选 B”的表、两边各自的典型题，以及参考资料。

<details>
<summary><strong>每一张模式决策页</strong></summary>

| # | 决策页 |
|---|---|
| P1 | [递归与迭代：迭代什么时候才重要](content/zh/dsa/patterns/P-01-recursion-vs-iteration.md) |
| P2 | [记忆化与制表，自顶向下与自底向上](content/zh/dsa/patterns/P-02-memoization-vs-tabulation.md) |
| P3 | [哈希表 vs 有序表：平均 O(1) vs 有序 O(log n)](content/zh/dsa/patterns/P-03-hash-map-vs-sorted-map.md) |
| P4 | [数组 vs 链表：各自何时胜出](content/zh/dsa/patterns/P-04-array-vs-linked-list.md) |
| P5 | [栈、队列与双端队列：选对 LIFO/FIFO](content/zh/dsa/patterns/P-05-stack-vs-queue-vs-deque.md) |
| P6 | [堆 vs 有序数组 vs 平衡 BST vs 语言内置有序集合](content/zh/dsa/patterns/P-06-heap-vs-sorted-array-vs-bst.md) |
| P6 | [双堆中位数原型：一个堆不够用的时候](content/zh/dsa/patterns/P-06b-two-heap-median-archetypes.md) |
| P6 | [K 路归并与游标堆](content/zh/dsa/patterns/P-06c-k-way-merge-cursor-heap.md) |
| P7 | [图上的 BFS 与 DFS：各自何时才是对的](content/zh/dsa/patterns/P-07-bfs-vs-dfs-graphs.md) |
| P8 | [Dijkstra vs Bellman-Ford vs 0-1 BFS vs A*：最短路径决策树](content/zh/dsa/patterns/P-08-shortest-paths-decision-tree.md) |
| P9 | [Kruskal vs Prim：最小生成树怎么选](content/zh/dsa/patterns/P-09-kruskal-vs-prim.md) |
| P10 | [快速排序 vs 归并排序 vs 堆排序：选对排序算法](content/zh/dsa/patterns/P-10-quicksort-vs-mergesort-vs-heapsort.md) |
| P11 | [二分查找 vs 线性扫描，以及答案上的二分查找](content/zh/dsa/patterns/P-11-binary-search-vs-linear-scan.md) |
| P12 | [双指针与滑动窗口：貌合神离的两类模式](content/zh/dsa/patterns/P-12-two-pointers-vs-sliding-window.md) |
| P13 | [迭代 vs Morris vs 递归树遍历](content/zh/dsa/patterns/P-13-iterative-vs-morris-vs-recursive-traversal.md) |
| P14 | [Trie vs 哈希表 vs 有序集合：前缀查询的决策](content/zh/dsa/patterns/P-14-trie-vs-hash-map-vs-sorted-set.md) |
| P15 | [连通性问题：Union-Find 还是 DFS](content/zh/dsa/patterns/P-15-union-find-vs-dfs.md) |
| P16 | [拓扑排序：Kahn 与基于 DFS 的版本](content/zh/dsa/patterns/P-16-topological-sort-kahn-vs-dfs.md) |
| P17 | [贪心 vs DP：贪心何时可被证明正确](content/zh/dsa/patterns/P-17-greedy-vs-dp.md) |
| P18 | [回溯 vs DP：子问题是否重叠](content/zh/dsa/patterns/P-18-backtracking-vs-dp.md) |
| P19 | [递归 vs DP：什么时候没有重叠（树遍历），什么时候有](content/zh/dsa/patterns/P-19-recursion-vs-dp.md) |
| P20 | [何时该动用位运算](content/zh/dsa/patterns/P-20-when-bit-manipulation.md) |
| P21 | [树上的 BFS 与 DFS：层序与中/前/后序语义](content/zh/dsa/patterns/P-21-bfs-vs-dfs-trees.md) |
| P22 | [求 top-K：Quickselect 与堆的取舍](content/zh/dsa/patterns/P-22-quickselect-vs-heap-top-k.md) |
| P23 | [滑动窗口原型：五种形态怎么区分](content/zh/dsa/patterns/P-23-sliding-window-archetypes.md) |
| P24 | [双指针原型：对撞、快慢、分区、原地去重、尾部归并](content/zh/dsa/patterns/P-24-two-pointer-archetypes.md) |
| P25 | [前缀和原型（含前缀和 + 哈希、前缀异或、差分数组、二维）](content/zh/dsa/patterns/P-25-prefix-sum-archetypes.md) |
| P26 | [单调栈原型](content/zh/dsa/patterns/P-26-monotonic-stack-archetypes.md) |
| P27 | [单调双端队列原型](content/zh/dsa/patterns/P-27-monotonic-deque-archetypes.md) |
| P28 | [链表分组的就地反转](content/zh/dsa/patterns/P-28-in-place-linked-list-reversal.md) |
| P29 | [循环排序原型：在 [1..n] 排列形态数组中的缺失、重复与首个正数](content/zh/dsa/patterns/P-29-cyclic-sort-archetypes.md) |
| P30 | [用堆或快速选择求 Top-K](content/zh/dsa/patterns/P-30-top-k-heap-or-quickselect.md) |
| P31 | [合并区间原型](content/zh/dsa/patterns/P-31-merge-intervals-archetypes.md) |
| P32 | [树形 DP 原型](content/zh/dsa/patterns/P-32-tree-dp-archetypes.md) |
| P33 | [位掩码 DP 原型](content/zh/dsa/patterns/P-33-bitmask-dp-archetypes.md) |
| P34 | [回溯模板 + 剪枝](content/zh/dsa/patterns/P-34-backtracking-template-pruning.md) |
| P35 | [扫描线原型](content/zh/dsa/patterns/P-35-sweep-line-archetypes.md) |

</details>

### 长篇题解

把经典面试题当完整文章来写：从暴力到最优的每一种解法、正确性证明、边界情况，以及追问变体。

<details>
<summary><strong>每一篇题解</strong></summary>

| LeetCode | 题解 | 逐步演示组件 |
|---|---|---|
| 1 | [两数之和](content/zh/dsa/editorials/LC-001-two-sum.md) | [Two Sum（LC 1）—— 最优的一趟哈希表](content/dsa/widgets/e-LC001-two-sum.yml) |
| 3 | [无重复字符的最长子串](content/zh/dsa/editorials/LC-003-longest-substring-without-repeating-characters.md) | [Longest Substring Without Repeating Characters (LC 3) — 最近索引跳跃](content/dsa/widgets/e-LC003-longest-substring.yml) |
| 5 | [最长回文子串](content/zh/dsa/editorials/LC-005-longest-palindromic-substring.md) | [最长回文子串（LC 5）—— 中心扩展](content/dsa/widgets/e-LC005-longest-palindrome.yml) |
| 11 | [盛最多水的容器](content/zh/dsa/editorials/LC-011-container-with-most-water.md) | [Container With Most Water（LC 11）—— 贪心双指针收缩](content/dsa/widgets/e-LC011-container.yml) |
| 15 | [三数之和](content/zh/dsa/editorials/LC-015-3sum.md) | [3Sum（LC 15）—— 排序 + 锚点 + 两端相向并去重](content/dsa/widgets/e-LC015-3sum.yml) |

</details>

### 交互组件规格

DSA 的重点章节配了逐步演示动画，比如滑动窗口、单调栈和 Dijkstra。每个动画都由 [`content/dsa/widgets/`](content/dsa/widgets/) 下的一份 YAML 规范描述，并在 [`_widget-registry.yml`](content/dsa/_widget-registry.yml) 里建了索引。在 GitHub 上，章节里的组件提示行会链到那份规范。

题解配套的组件和上面的[题解](#长篇题解)列在一起。

<details>
<summary><strong>每一个模式组件</strong> &nbsp;—&nbsp; 以及用到它的章节</summary>

| 组件 | 用于 |
|---|---|
| [递归调用栈（阶乘；三种形状对比）](content/dsa/widgets/w-01-recursion-call-stack.yml) | [递归的心智模型](content/zh/dsa/part-0-foundations/02-recursion-mental-model.md), [递归模式：线性、树形与分治](content/zh/dsa/part-7-recursion-backtracking/00-recursion-patterns.md) |
| [哈希表（链式插入 + 查找）](content/dsa/widgets/w-02-hash-table.yml) | [哈希映射与哈希集合](content/zh/dsa/part-1-linear-data-structures/03-hash-maps.md), [哈希冲突与装载因子](content/zh/dsa/part-1-linear-data-structures/04-hash-collisions.md) |
| [矩阵旋转（分层四循环）](content/dsa/widgets/w-03-matrix-rotation.yml) | [矩阵操作](content/zh/dsa/part-1-linear-data-structures/07-matrix-manipulation.md) |
| [二分查找（精确匹配 + 旋转数组）](content/dsa/widgets/w-04-binary-search.yml) | [二分查找：标准版本](content/zh/dsa/part-2-search-sort/01-binary-search-canonical.md), [二分查找的变体：lower_bound、upper_bound、峰值与旋转数组](content/zh/dsa/part-2-search-sort/02-binary-search-variants.md) |
| [排序可视化（归并排序 + 快速排序）](content/dsa/widgets/w-05-sorting-visualizer.yml) | [比较排序 I：插入排序与归并排序](content/zh/dsa/part-2-search-sort/03-comparison-sorts-1.md), [比较排序 II：快速排序、分区与生产级混合排序](content/zh/dsa/part-2-search-sort/04-comparison-sorts-2.md) |
| [快速排序分区（Lomuto，逐步演示）](content/dsa/widgets/w-06-quicksort-partition.yml) | [比较排序 II：快速排序、分区与生产级混合排序](content/zh/dsa/part-2-search-sort/04-comparison-sorts-2.md) |
| [快速选择：分区，然后只递归一侧](content/dsa/widgets/w-07-quickselect.yml) | [快速选择：线性时间选择](content/zh/dsa/part-2-search-sort/07-quickselect.md) |
| [双指针（和为目标值）](content/dsa/widgets/w-08-two-pointers-3sum.yml) | [双指针：两端相向](content/zh/dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md) |
| [滑动窗口（固定 + 变长）](content/dsa/widgets/w-09-sliding-window-expansion.yml) | [滑动窗口：固定长度](content/zh/dsa/part-3-pointers-window-prefix/02-sliding-window-fixed.md), [滑动窗口：变长窗口](content/zh/dsa/part-3-pointers-window-prefix/03-sliding-window-variable.md) |
| [前缀和（累积 + 区间查询）](content/dsa/widgets/w-10-prefix-sum-cumulative.yml) | [前缀和与差分数组](content/zh/dsa/part-3-pointers-window-prefix/04-prefix-sums.md) |
| [单调栈（下一个更大元素扫描）](content/dsa/widgets/w-11-monotonic-stack.yml) | [单调栈](content/zh/dsa/part-4-stack-queue-patterns/00-monotonic-stack.md) |
| [用两个栈做均摊队列（LC 232）](content/dsa/widgets/w-12-amortized-queue-via-stacks.yml) | [用栈实现队列](content/zh/dsa/part-4-stack-queue-patterns/04-queue-from-stacks.md) |
| [单调双端队列（滑动窗口最大值）](content/dsa/widgets/w-12-monotonic-deque.yml) | [单调双端队列](content/zh/dsa/part-4-stack-queue-patterns/01-monotonic-deque.md) |
| [链表指针重连](content/dsa/widgets/w-13-linked-list-pointer-rewiring.yml) | [链表基础：哨兵、指针重连与双向链表设计](content/zh/dsa/part-5-linked-lists/00-linked-list-fundamentals.md), [反转模式](content/zh/dsa/part-5-linked-lists/01-reversal-patterns.md), [每 k 个一组反转](content/zh/dsa/part-5-linked-lists/02-reverse-in-groups-k.md) |
| [Floyd 龟兔算法（环检测）](content/dsa/widgets/w-14-floyd-cycle.yml) | [环检测（Floyd 龟兔算法）](content/zh/dsa/part-5-linked-lists/03-cycle-detection.md) |
| [LRU 缓存（哈希映射 + 双向链表）](content/dsa/widgets/w-15-lru-cache.yml) | [LRU 缓存：哈希映射 + 双向链表](content/zh/dsa/part-5-linked-lists/05-lru-cache.md), [LRU 缓存：设计框架、并发与多级部署](content/zh/dsa/part-13-design-the-data-structure/00-lru-cache.md) |
| [树遍历动画（DFS 前序 / 中序 / 后序 + BFS 层序）](content/dsa/widgets/w-16-tree-traversal-animator.yml) | [树的遍历：前序、中序、后序](content/zh/dsa/part-6-trees-heaps/01-tree-traversals.md), [层序遍历：树上的 BFS](content/zh/dsa/part-6-trees-heaps/03-level-order-traversal.md) |
| [Morris 中序遍历（线索化）](content/dsa/widgets/w-17-morris-thread.yml) | [Morris 遍历：用线索实现 O(1) 空间的中序遍历](content/zh/dsa/part-6-trees-heaps/02-morris-traversal.md) |
| [堆操作：push 后的上滤、pop 后的下滤](content/dsa/widgets/w-18-heap-operations.yml) | [堆与优先队列](content/zh/dsa/part-6-trees-heaps/04-heaps-priority-queues.md) |
| [BST 操作与 AVL 旋转](content/dsa/widgets/w-19-bst-rotations.yml) | [二叉搜索树](content/zh/dsa/part-6-trees-heaps/05-binary-search-trees.md), [AVL 树与旋转](content/zh/dsa/part-6-trees-heaps/06-avl-rotations.md) |
| [Trie — 插入 + 前缀遍历 + 频率自动补全](content/dsa/widgets/w-20-trie.yml) | [Trie](content/zh/dsa/part-6-trees-heaps/08-tries.md), [Trie 自动补全](content/zh/dsa/part-13-design-the-data-structure/04-trie-autocomplete.md) |
| [线段树（构建 / 更新 / 查询）](content/dsa/widgets/w-21-segment-tree.yml) | [线段树](content/zh/dsa/part-6-trees-heaps/10-segment-tree.md) |
| [回溯决策树](content/dsa/widgets/w-22-backtracking-tree.yml) | [回溯模板](content/zh/dsa/part-7-recursion-backtracking/01-backtracking-template.md), [子集、组合、排列](content/zh/dsa/part-7-recursion-backtracking/02-subsets-combinations-permutations.md) |
| [N-Queens —— 在 n=4 上放置 / 攻击 / 回退](content/dsa/widgets/w-23-n-queens.yml) | [N-Queens：剪枝与约束传播](content/zh/dsa/part-7-recursion-backtracking/03-n-queens.md) |
| [数独求解（约束传播 + MRV）](content/dsa/widgets/w-24-sudoku-grid.yml) | [数独求解：约束传播与前向检查](content/zh/dsa/part-7-recursion-backtracking/04-sudoku-solver.md) |
| [Graph BFS — 前沿扩张（网格上的多源）](content/dsa/widgets/w-25-graph-bfs.yml) | [广度优先搜索](content/zh/dsa/part-8-graphs/01-bfs.md) |
| [DFS 探索 —— 5×5 网格上从边界发起的迭代 DFS](content/dsa/widgets/w-26-graph-dfs.yml) | [深度优先搜索](content/zh/dsa/part-8-graphs/02-dfs.md) |
| [拓扑排序（Kahn 对比 DFS 后序）](content/dsa/widgets/w-27-topological-sort-kahn.yml) | [拓扑排序：Kahn 算法](content/zh/dsa/part-8-graphs/04-topological-sort-kahn.md), [拓扑排序：DFS 后序反转](content/zh/dsa/part-8-graphs/05-topological-sort-dfs.md) |
| [Union-Find：带路径压缩与按秩合并的父指针森林](content/dsa/widgets/w-28-union-find.yml) | [Union-Find：父指针森林、路径压缩与按秩合并](content/zh/dsa/part-8-graphs/08-union-find.md) |
| [Dijkstra 最短路径算法](content/dsa/widgets/w-29-dijkstra.yml) | [Dijkstra 最短路径算法](content/zh/dsa/part-8-graphs/09-dijkstra.md) |
| [最小生成树：Kruskal 与 Prim 并排对比](content/dsa/widgets/w-30-mst-kruskal-prim.yml) | [最小生成树：Kruskal 算法](content/zh/dsa/part-8-graphs/11-mst-kruskal.md), [最小生成树：Prim 算法](content/zh/dsa/part-8-graphs/12-mst-prim.md) |
| [DP 表格填充](content/dsa/widgets/w-31-dp-table-fill.yml) | [动态规划：从递归到记忆化](content/zh/dsa/part-9-dynamic-programming/00-dp-recursion-to-memo.md), [DP：自底向上制表](content/zh/dsa/part-9-dynamic-programming/01-dp-bottom-up-tabulation.md), [最长公共子序列](content/zh/dsa/part-9-dynamic-programming/06-lcs.md), [编辑距离](content/zh/dsa/part-9-dynamic-programming/07-edit-distance.md) |
| [背包 DP 填表（0/1 对比无界）](content/dsa/widgets/w-32-knapsack-fill.yml) | [0/1 背包](content/zh/dsa/part-9-dynamic-programming/04-knapsack-01.md), [无界背包：物品可以被反复选择](content/zh/dsa/part-9-dynamic-programming/05-knapsack-unbounded.md) |
| [用耐心排序求 LIS](content/dsa/widgets/w-33-lis-patience.yml) | [最长递增子序列：耐心排序](content/zh/dsa/part-9-dynamic-programming/09-lis-patience-sort.md) |
| [位掩码 BFS：(node, mask) 状态扩展](content/dsa/widgets/w-34-bitmask-dp.yml) | [位掩码 DP](content/zh/dsa/part-9-dynamic-programming/14-bitmask-dp.md) |
| [前缀和 + 哈希表组合](content/dsa/widgets/w-35-prefix-sum-hash-combo.yml) | [前缀和 + 哈希表组合](content/zh/dsa/part-3-pointers-window-prefix/05-prefix-sum-hash.md) |
| [区间调度：不重叠与最少箭数](content/dsa/widgets/w-36-interval-scheduling.yml) | [区间调度：比较器就是算法](content/zh/dsa/part-10-greedy/01-interval-scheduling.md) |
| [Bellman-Ford 负环检测](content/dsa/widgets/w-37-bellman-ford.yml) | [Bellman-Ford 与负环](content/zh/dsa/part-8-graphs/10-bellman-ford.md) |
| [KMP 的失败函数构建与匹配阶段](content/dsa/widgets/w-38-kmp.yml) | [KMP 与失败函数](content/zh/dsa/part-12-strings-pattern-matching/02-kmp.md) |
| [霍夫曼编码（小顶堆 + 建树 + 码字遍历）](content/dsa/widgets/w-39-huffman-encoding.yml) | [霍夫曼编码](content/zh/dsa/part-10-greedy/03-huffman-encoding.md) |
| [Z 数组构造与 [l, r) 匹配窗口](content/dsa/widgets/w-40-z-array.yml) | [Z 算法](content/zh/dsa/part-12-strings-pattern-matching/03-z-algorithm.md) |

</details>

---

## The AI Engineering Handbook

**如何在模型 API 之上构建并运营生产级 AI 产品。**

[网站](https://ai-eng.handbook.academy) · [`content/ai-eng/`](content/ai-eng/) · [本书导读](content/zh/ai-eng/README.md) · [AI Engineering 风格规则](STYLE_GUIDE.md#ai-engineering-specific-deviations)

AI 工程师在基础模型之上构建产品，而不训练模型。本书始终停在模型 API 的产品侧：模型如何表现、如何可靠地调用、如何撰写并版本化提示词、如何评估输出、如何拼装上下文、如何检索、如何构建 agent，以及在生产中如何观测、防护并为这一切付费。

**第一次读请按顺序读。** 第 0 部分给出了全书遵循的升级阶梯：先提示词，再上下文，再检索，再工具，然后才是 agent，微调放在最后。**讲评估的第 4 部分故意排在检索与 agent 之前**，因为它之后的每一章都假设你能衡量一次改动是否真的有用。

**一章是怎么写成的。** 没有固定模板。多数章节从一个真实事故或一个失败案例开场，讲清机制，最后落在一件你能直接用的东西上：一份检查清单、一条决策规则，或一段能跑起来的代码片段。代码用 Python。插图放在每个部分的 `images/` 目录下。价格、模型名和基准分数都会标注“截至”日期与来源。

**它与 HLD 第 9 部分的关系。** [HLD 第 9 部分](#hld-part-9)是白板视角：如何在规模上设计一个 AI 系统。本书是编辑器视角：如何把它建起来并跑起来。两本书互相链接。

### AI Engineering 各部分速览

点一个部分编号跳到它的章节，或者展开下面任意一个部分。

| 部分 | 内容简介 | 难度 |
|---|---|---|
| [0](#ai-eng-part-0) | **从这里开始** —— AI 工程师是什么、软件工程基本功、概率系统思维、升级阶梯 | 入门 |
| [1](#ai-eng-part-1) | **模型的行为** —— 模型档位、token、采样、向量嵌入、推理模型、成本-质量-延迟三角 | 入门到中级 |
| [2](#ai-eng-part-2) | **与模型 API 打交道** —— 流式、结构化输出、工具调用、重试、网关、提示词缓存 | 入门到中级 |
| [3](#ai-eng-part-3) | **提示词工程** —— 提示词结构、少样本、思维链、模板、版本管理 | 入门到中级 |
| [4](#ai-eng-part-4) | **评测基础** —— 先看数据、第一个评测集、把断言放进 CI | 入门 |
| [5](#ai-eng-part-5) | **上下文工程** —— 上下文组装、压缩、记忆、工具元数据、上下文测试 | 中级 |
| [6](#ai-eng-part-6) | **检索（RAG）** —— 分块、向量与混合检索、重排、引用、检索评测、生产环境索引 | 中级到进阶 |
| [7](#ai-eng-part-7) | **智能体工程** —— 工作流与智能体、工具设计、MCP、记忆、人在回路、多智能体、语音、框架选型 | 中级到进阶 |
| [8](#ai-eng-part-8) | **评测与可观测性** —— LLM 作为评判、评判偏差、RAG 与智能体评测、CI 门禁、A/B 测试、链路追踪、看板 | 中级 |
| [9](#ai-eng-part-9) | **安全与治理** —— 提示注入、致命三要素、护栏、滥用、PII、合规、红队测试 | 中级到进阶 |
| [10](#ai-eng-part-10) | **生产级 AI** —— 托管与自托管、微调决策、成本与延迟工程、灰度发布、SLO | 中级 |
| [11](#ai-eng-part-11) | **AI 产品工程** —— AI 用户体验、失败恢复、产品指标、实验、定价、反馈回路 | 中级到进阶 |

<details>
<summary><a id="ai-eng-part-0"></a><strong>第 0 部分 · 从这里开始</strong></summary>

**级别：** 入门 &nbsp;·&nbsp; **适合：** 所有人，包括刚接触 LLM 开发的经验丰富的工程师。

AI 工程师是什么，软件工程基本功，用概率系统的思维看问题，以及全书遵循的升级阶梯：提示词、上下文、检索、工具、agent、微调。

| # | 章节 | 内容简介 |
|---|---|---|
| 0.0 | [AI 工程师是什么（以及不是什么）](content/zh/ai-eng/part-0-start-here/00-what-an-ai-engineer-is.md) | AI 工程师、ML 工程师与研究员的区别：API 分界线、三种角色，以及这本手册到底教什么、跳过什么。 |
| 0.1 | [软件基线](content/zh/ai-eng/part-0-start-here/01-the-software-baseline.md) | 本书默认你已经具备的七项工程技能（Python、HTTP、JSON、异步、Git、Docker、Postgres），它们对 AI 工作各自为何重要，以及该去哪儿补。 |
| 0.2 | [在概率系统中思考](content/zh/ai-eng/part-0-start-here/02-thinking-in-probabilistic-systems.md) | 为什么同一个 LLM 调用即使在 temperature 0 下也会返回不同答案、五种失效模式，以及为什么优雅降级是默认姿态。 |
| 0.3 | [升级阶梯](content/zh/ai-eng/part-0-start-here/03-the-escalation-ladder.md) | 整本书都跑在这套优先级顺序上：提示词、上下文、检索、工具、Agent、微调。哪一阶最便宜又能修好这次失败，就停在哪一阶。 |

</details>

<details>
<summary><a id="ai-eng-part-1"></a><strong>第 1 部分 · 模型如何表现</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 任何调用模型 API、想预判账单、延迟和失败模式的人。

模型版图、token 与上下文窗口、采样、embedding、推理模型、成本-质量-延迟三角，以及刚好够做出正确系统决策的那点 ML 知识。

| # | 章节 | 内容简介 |
|---|---|---|
| 1.0 | [模型全景图](content/zh/ai-eng/part-1-how-models-behave/00-the-model-landscape.md) | 把任何模型放进对应档位，并知道每一档是干什么的：前沿、中端、快速、本地，外加开放权重与托管、对话与推理与多模态之分。 |
| 1.1 | [Token 与上下文窗口](content/zh/ai-eng/part-1-how-models-behave/01-tokens-and-context-windows.md) | Token 是 LLM 计费、限额和提示词的计量单位。如何在发送前用 Python 数清 token，以及非拉丁文字为何更贵。 |
| 1.2 | [采样与 temperature](content/zh/ai-eng/part-1-how-models-behave/02-sampling-and-temperature.md) | temperature 和 top_p 如何重塑模型的概率分布、如何按任务挑选取值，以及为什么 temperature 0 并不等于确定性保证。 |
| 1.3 | [嵌入向量（Embeddings）](content/zh/ai-eng/part-1-how-models-behave/03-embeddings.md) | 什么是嵌入、为什么归一化向量上的余弦相似度与点积等价，以及建立在嵌入之上的四种工程模式：搜索、记忆、路由、聚类。 |
| 1.4 | [推理模型](content/zh/ai-eng/part-1-how-models-behave/04-reasoning-models.md) | 什么是测试时计算，各家厂商的力度与预算旋钮，为什么你要为隐形的思考 token 付费，以及推理在什么时候值这个价。 |
| 1.5 | [成本-质量-延迟三角](content/zh/ai-eng/part-1-how-models-behave/05-the-cost-quality-latency-triangle.md) | 全书反复回到的心智模型：成本、质量、延迟是三个不可能同时拉满的维度。先说出你牺牲哪一个，再去学那些能绕过它的杠杆。 |
| 1.6 | [够用的机器学习](content/zh/ai-eng/part-1-how-models-behave/06-just-enough-ml.md) | 一个从不训练模型的 AI 工程师仍然需要的三个机器学习概念：数据划分、过拟合、分布漂移。把它们当作评估决策来理解。 |

</details>

<details>
<summary><a id="ai-eng-part-2"></a><strong>第 2 部分 · 与模型 API 打交道</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 第一次把 LLM 功能接进真实服务的工程师。

可靠地调用模型：流式输出、结构化输出、工具调用、错误与兜底、模型选型、网关与路由，以及提示词缓存的经济账。

| # | 章节 | 内容简介 |
|---|---|---|
| 2.0 | [调用模型](content/zh/ai-eng/part-2-working-with-model-apis/00-calling-models.md) | 一次 chat completion 的 POST 请求背后发生了什么：消息、角色、系统提示词，以及你必须并排看清的三家 provider 请求形态。 |
| 2.1 | [流式输出](content/zh/ai-eng/part-2-working-with-model-apis/01-streaming.md) | 逐 token 的流式输出如何在感知延迟上取胜，它为什么会破坏输出校验，以及同时拿到两者的四种做法。 |
| 2.2 | [结构化输出](content/zh/ai-eng/part-2-working-with-model-apis/02-structured-outputs.md) | 为什么 LLM 与消费其输出的代码之间的契约是 schema 强制的 JSON 而不是自由文本，以及如何用 Pydantic 把它接起来。 |
| 2.3 | [函数调用与工具调用](content/zh/ai-eng/part-2-working-with-model-apis/03-function-and-tool-calling.md) | 模型提出一次工具调用，你来执行，模型接着往下走：怎么定义工具 schema、怎么跑这个循环、怎么并行，以及怎么返回模型能据以行动的错误。 |
| 2.4 | [错误、重试与回退](content/zh/ai-eng/part-2-working-with-model-apis/04-errors-retries-fallbacks.md) | 如何应对每个 LLM API 都会抛给你的超时、限流与过载响应，配上一套正确的退避、回退与幂等规则。 |
| 2.5 | [如何选模型与读懂基准测试](content/zh/ai-eng/part-2-working-with-model-apis/05-choosing-models-and-reading-benchmarks.md) | 公开榜单预测不了你的产品质量。按档位筛出候选、用自己的评测集实测、然后定下来。整个流程就这三步。 |
| 2.6 | [网关与路由](content/zh/ai-eng/part-2-working-with-model-apis/06-gateways-and-routing.md) | 用一个 API 挡在所有供应商前面，先便宜后昂贵的级联，供应商故障切换，以及判断网关何时才值回运维成本的那条准则。 |
| 2.7 | [提示词缓存](content/zh/ai-eng/part-2-working-with-model-apis/07-prompt-caching.md) | API 原生 LLM 产品里最大的那根成本杠杆。一个 5 万 token 的 agent 跑 10 轮，改一处配置，账单从 $1.53 掉到 $0.47。 |

</details>

<details>
<summary><a id="ai-eng-part-3"></a><strong>第 3 部分 · 提示词工程</strong></summary>

**级别：** 入门到中级 &nbsp;·&nbsp; **适合：** 任何负责一条生产提示词的人。

生产环境里的提示词：结构拆解、few-shot、思维链与推理模型带来的反转、模板与串联，以及可回滚的提示词版本管理。

| # | 章节 | 内容简介 |
|---|---|---|
| 3.0 | [生产级提示词的解剖](content/zh/ai-eng/part-3-prompt-engineering/00-anatomy-of-a-production-prompt.md) | 生产系统提示词必备的五个部分，单体式上帝提示词为何失效，以及如何布置防御性指令来对抗提示词注入。 |
| 3.1 | [零样本与少样本](content/zh/ai-eng/part-3-prompt-engineering/01-zero-shot-and-few-shot.md) | 示例何时值回它占的 token，何时不值。判断规则、过度提示的悬崖，以及提示词缓存如何改写这笔账。 |
| 3.2 | [思维链、ReAct 与自一致性](content/zh/ai-eng/part-3-prompt-engineering/02-chain-of-thought-react-self-consistency.md) | 从 2023 年的提示词文献里活下来的三项技术，以及推理模型带来的规则反转：各自何时该用，何时该删掉。 |
| 3.3 | [模板与链式调用](content/zh/ai-eng/part-3-prompt-engineering/03-templates-and-chaining.md) | 把提示词当成有类型的代码来写，以及你每周都要做的那次取舍：一个大提示词，还是几个小提示词串成链。 |
| 3.4 | [提示词版本管理](content/zh/ai-eng/part-3-prompt-engineering/04-prompt-versioning.md) | 把提示词改动当作代码改动来对待：注册表、版本钉定、回滚与评审，外加一套一周就能搭好的最小方案。 |

</details>

<details>
<summary><a id="ai-eng-part-4"></a><strong>第 4 部分 · 评估基础</strong></summary>

**级别：** 入门 &nbsp;·&nbsp; **适合：** 所有人。如果只读一个部分，就读这个。

最小可用的评估闭环，故意排在 RAG 和 agent 之前：看你的数据，建起第一个评估集，把断言接进 CI。这个部分之后的任何内容，没有评估都不算能上线。

| # | 章节 | 内容简介 |
|---|---|---|
| 4.0 | [为什么没有评测就上不了线](content/zh/ai-eng/part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md) | 凭感觉开发的失效模式、把评测当作 AI 工程师的 CI，以及让你脱离打地鼠循环的成熟度阶梯。 |
| 4.1 | [先看你的数据](content/zh/ai-eng/part-4-eval-fundamentals/01-look-at-your-data.md) | LLM 评测里投入产出比最高的一个习惯：在自动化任何东西之前，先手工读 trace，建出一套失败分类体系。 |
| 4.2 | [你的第一套评估集](content/zh/ai-eng/part-4-eval-fundamentals/02-your-first-eval-set.md) | 一周之内搭起一套 50 条样本的评估集：用真实或足够逼真的输入，打二元的通过/不通过标签，放进 git 做版本管理，再从线上失败中不断长大。 |
| 4.3 | [LLM 输出的断言与单元测试](content/zh/ai-eng/part-4-eval-fundamentals/03-assertions-and-unit-tests.md) | 在任何 judge 模型介入之前，每个 LLM 提交都该先过一遍的那些便宜、确定性的检查：schema、contains、regex、代码能跑、SQL 能解析，接进 pytest 和 CI。 |

</details>

<details>
<summary><a id="ai-eng-part-5"></a><strong>第 5 部分 · 上下文工程</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 提示词已经长成流水线的工程师。

构建模型看到的完整世界：上下文的拼装与路由、压缩与预算、记忆与状态、工具上下文，以及测试真正被拼进去的内容。

| # | 章节 | 内容简介 |
|---|---|---|
| 5.0 | [上下文不只是提示词](content/zh/ai-eng/part-5-context-engineering/00-context-is-more-than-the-prompt.md) | 每一轮进入上下文窗口的六样东西，为什么提示词工程只覆盖其中一样，以及上下文工程这门手艺究竟负责什么。 |
| 5.1 | [上下文组装与路由](content/zh/ai-eng/part-5-context-engineering/01-context-assembly-and-routing.md) | 为什么 100 万 token 的窗口不等于免费的上下文，以及先路由、再组装这条流水线如何决定模型真正看到什么。 |
| 5.2 | [压缩与上下文预算](content/zh/ai-eng/part-5-context-engineering/02-compression-and-context-budgets.md) | 上下文更多不等于更好。中间遗忘（lost-in-the-middle）与上下文腐化（context rot）如何拖累长提示，以及何时该裁剪、摘要还是重新检索。 |
| 5.3 | [记忆与状态](content/zh/ai-eng/part-5-context-engineering/03-memory-and-state.md) | LLM 在多轮之间、会话之间、乃至数年之间究竟记得什么，以及让每种作用域都管用、又不挤爆上下文窗口的那些技术。 |
| 5.4 | [工具上下文与元数据](content/zh/ai-eng/part-5-context-engineering/04-tool-context-and-metadata.md) | 模型没有时钟，没有角色意识，也没有区域设置的概念。你注入的元数据、以及你描述工具的方式，能挡住好几类自信地答错的回答。 |
| 5.5 | [测试与观测上下文](content/zh/ai-eng/part-5-context-engineering/05-testing-and-observing-context.md) | 把组装好的上下文做成快照，对组装逻辑做回归测试，并采集能回答“模型为什么看到这些内容”的 traces（追踪）。 |

</details>

<details>
<summary><a id="ai-eng-part-6"></a><strong>第 6 部分 · 检索（RAG）</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 任何要在自己的文档上做搜索或问答的人。

把检索做对：什么时候真的需要 RAG、分块、向量与混合检索、重排、查询理解、引用、检索评估，以及生产环境的索引。

| # | 章节 | 内容简介 |
|---|---|---|
| 6.0 | [到底什么时候才真的需要 RAG](content/zh/ai-eng/part-6-retrieval-augmented-generation/00-when-you-actually-need-rag.md) | 把 RAG、长上下文、微调当成一道成本题：长上下文按查询付费，RAG 建一次索引反复用。默认规则、例外情况，以及 RAG 反而是错答案的时候。 |
| 6.1 | [文档处理与分块](content/zh/ai-eng/part-6-retrieval-augmented-generation/01-document-processing-and-chunking.md) | 解析方式与分块边界如何决定你的检索上限，为什么重叠和块大小都是评测问题，以及哪些默认值能扛住真实语料的检验。 |
| 6.2 | [向量嵌入与向量检索](content/zh/ai-eng/part-6-retrieval-augmented-generation/02-embeddings-and-vector-search.md) | 怎么用自己的数据挑嵌入模型，为什么 pgvector 是默认的向量存储，HNSW 和 IVFFlat 到底在做什么，以及专用向量数据库真正值回票价的那条诚实阈值线在哪。 |
| 6.3 | [混合检索与重排序](content/zh/ai-eng/part-6-retrieval-augmented-generation/03-hybrid-search-and-reranking.md) | 为什么在精确词项上 BM25 仍然胜过向量检索、如何用 RRF 把两者融合，以及重排序模型什么时候才配得上它带来的延迟。 |
| 6.4 | [查询理解](content/zh/ai-eng/part-6-retrieval-augmented-generation/04-query-understanding.md) | 改写、多查询、问题分解与路由：弥合用户敲进去的措辞和检索器能匹配上的内容之间的落差。 |
| 6.5 | [引用与依据](content/zh/ai-eng/part-6-retrieval-augmented-generation/05-citations-and-grounding.md) | 让 RAG 的回答可以被验证：片段级引用（span-level citation）、检索为空时直接拒答，以及由忠实度与流畅度之争决定的那两件事。 |
| 6.6 | [高级检索](content/zh/ai-eng/part-6-retrieval-augmented-generation/06-advanced-retrieval.md) | 三种在特定查询上胜出原生 RAG 的模式：智能体式检索（agentic retrieval）、GraphRAG、视觉原生 PDF 检索。各自何时值得那份复杂度，何时不值得。 |
| 6.7 | [评估检索](content/zh/ai-eng/part-6-retrieval-augmented-generation/07-evaluating-retrieval.md) | RAG 评测主要是检索评测。用大白话讲清 recall@k 与 MRR、如何从真实查询构建评测集，以及把检索失败与生成失败区分开的 2×2 矩阵。 |
| 6.8 | [生产级 RAG](content/zh/ai-eng/part-6-retrieval-augmented-generation/08-production-rag.md) | 把 RAG 原型变成可运维系统的五个运维关注点：增量索引、重建索引税、多租户隔离、元数据过滤，以及分阶段延迟预算。 |

</details>

<details>
<summary><a id="ai-eng-part-7"></a><strong>第 7 部分 · Agent 工程</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 正在判断要不要上 agent、并且真要动手建一个的工程师。

工作流与 agent 的分界、工具设计、MCP 及其安全模型、记忆、human-in-the-loop、长时间运行的 agent、多 agent 系统、agent 原型、语音 agent，以及框架怎么选。

| # | 章节 | 内容简介 |
|---|---|---|
| 7.0 | [Workflow 与 agent](content/zh/ai-eng/part-7-agent-engineering/00-workflow-vs-agent.md) | 控制流能提前用 Python 写出来，那它就是 workflow；如果由模型在运行时决定，那它就是 agent。生产环境里的大部分收益其实来自 workflow。 |
| 7.1 | [五种工作流模式](content/zh/ai-eng/part-7-agent-engineering/01-the-five-workflow-patterns.md) | 链式调用、路由、并行化、编排者-工作者、评估者-优化者：每一种都配了代码、一个用例，以及它在生产环境里真正咬人的失效模式。 |
| 7.2 | [Agent 循环](content/zh/ai-eng/part-7-agent-engineering/02-the-agent-loop.md) | 思考、行动、观察，以及那个没人愿意谈的部分：循环怎么结束。停止条件、真正有效的反思，还有为什么生产环境的 agent 跟演示里完全是两回事。 |
| 7.3 | [工具设计](content/zh/ai-eng/part-7-agent-engineering/03-tool-design.md) | 智能体工程的基本功：让非法调用无法发生的 schema、模型读不歪的描述、模型能据以恢复的错误，以及恰当的工具数量。 |
| 7.4 | [MCP：模型上下文协议](content/zh/ai-eng/part-7-agent-engineering/04-mcp-model-context-protocol.md) | 如何通过一个开放协议把工具、数据和提示词暴露给任意 AI 宿主；端到端搭出一个服务端和一个客户端。 |
| 7.5 | [MCP 安全](content/zh/ai-eng/part-7-agent-engineering/05-mcp-security.md) | MCP 带来的四类攻击——工具投毒、跨服务器遮蔽、上架后调包、借工具返回结果注入——以及真正能挡住它们的架构层防御。 |
| 7.6 | [智能体记忆与任务状态](content/zh/ai-eng/part-7-agent-engineering/06-agent-memory-and-task-state.md) | 智能体可以存放信息的四个地方，为什么大多数团队用错了地方，以及怎样设计出能在任务中途扛住 Pod 重启的状态。 |
| 7.7 | [人在回路（Human-in-the-loop）](content/zh/ai-eng/part-7-agent-engineering/07-human-in-the-loop.md) | 把审批门做成工具调用、按置信度升级、以及交接设计：要让有选择性的人工监督真正生效，这三件彼此独立的事你都得自己做。 |
| 7.8 | [长时间运行的智能体](content/zh/ai-eng/part-7-agent-engineering/08-long-running-agents.md) | 预算、急停开关、检查点、重放，以及同步与异步的取舍：让一个跑几分钟的智能体不至于烧掉一个月 API 预算的那部分运行框架工程。 |
| 7.9 | [多智能体系统](content/zh/ai-eng/part-7-agent-engineering/09-multi-agent-systems.md) | 多智能体拓扑什么时候值它那 15 倍的 token 成本，什么时候不值，以及那套能覆盖大多数生产工作的单写者模式。 |
| 7.10 | [Agent 原型](content/zh/ai-eng/part-7-agent-engineering/10-agent-archetypes.md) | 研究、编码、浏览器与 Computer Use 四类 agent 只在一件事上不同，而这件事会级联到其他所有方面：模型每一步看到什么。 |
| 7.11 | [语音智能体](content/zh/ai-eng/part-7-agent-engineering/11-voice-agents.md) | 定义语音智能体的 800 毫秒预算、级联与原生之间的权衡，以及为什么打断是整条链路里最难的正确性问题。 |
| 7.12 | [选择框架](content/zh/ai-eng/part-7-agent-engineering/12-choosing-a-framework.md) | 框架带来状态、重试与追踪。多数团队并不需要它们。裸 SDK、轻结构、完整图运行时三档的判断规则。 |

</details>

<details>
<summary><a id="ai-eng-part-8"></a><strong>第 8 部分 · 评估与可观测性</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 已经过了第一个评估集、需要在 CI 和生产中信任评估的团队。

全书的主干：LLM-as-judge 及其失效模式、RAG 与 agent 的分组件评估、CI 中的回归门禁、在线评估、追踪、监控，以及数据飞轮。

| # | 章节 | 内容简介 |
|---|---|---|
| 8.0 | [评测分类法](content/zh/ai-eng/part-8-evaluation-and-observability/00-the-eval-taxonomy.md) | 决定哪一类评测能回答你的问题的四个维度：离线与在线、组件与端到端、有参考答案与无参考答案、自动评分与评判者评分与人工评分。 |
| 8.1 | [LLM-as-judge](content/zh/ai-eng/part-8-evaluation-and-observability/01-llm-as-judge.md) | 构建一个你能信任的第二 LLM 评判者：用二值的通过/不通过取代李克特量表、提示词该怎么写，以及上线前必须跑通的人工一致性回路。 |
| 8.2 | [裁判的失败模式](content/zh/ai-eng/part-8-evaluation-and-observability/02-judge-failure-modes.md) | LLM-as-judge 流水线里的位置偏差、长度偏差、自我偏好与风格偏差，连带实测数字和生产里真正管用的缓解手段。 |
| 8.3 | [评估 RAG 与 agent](content/zh/ai-eng/part-8-evaluation-and-observability/03-evaluating-rag-and-agents.md) | 组件评测、工具选择准确率、轨迹评估，以及“最弱一环”模式——它告诉你复合系统里该修哪一块。 |
| 8.4 | [回归套件与 CI 门禁](content/zh/ai-eng/part-8-evaluation-and-observability/04-regression-suites-and-ci-gates.md) | 把评测接进 CI 当成真的合并门禁：两层黄金集、以差分阈值取代绝对地板，以及如何让抖动的概率性测试仍然可信。 |
| 8.5 | [在线评估与 A/B 测试](content/zh/ai-eng/part-8-evaluation-and-observability/05-online-evaluation-and-ab-testing.md) | 为什么你的离线评估谎报了一个 3 分的提升、哪些隐式信号不会撒谎，以及 LLM 的 A/B 测试到底需要多大样本量。 |
| 8.6 | [链路追踪](content/zh/ai-eng/part-8-evaluation-and-observability/06-tracing.md) | LLM 调用、检索与 agent 循环的 span：gen_ai.* 的 OpenTelemetry 约定，以及厂商私有追踪为何会把你锁死。 |
| 8.7 | [监控与仪表盘](content/zh/ai-eng/part-8-evaluation-and-observability/07-monitoring-and-dashboards.md) | 为什么 LLM 系统要把成本、延迟、质量和用量放在同一块屏上；怎么对质量回归和成本尖峰告警；2026 年该选哪家可观测性平台。 |
| 8.8 | [反馈与数据飞轮](content/zh/ai-eng/part-8-evaluation-and-observability/08-feedback-and-the-data-flywheel.md) | 生产流量如何复利成更好的评估集与更好的产品，而不是变成一轮接一轮、每个 sprint 都在修 bug 的跑步机。 |

</details>

<details>
<summary><a id="ai-eng-part-9"></a><strong>第 9 部分 · 安全与治理</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 任何要上线一个会触及用户数据或工具的 LLM 功能的人。

提示词注入、致命三元组与 agent 安全、护栏、防滥用、PII 与隐私、合规，以及对自己的系统做红队测试。

| # | 章节 | 内容简介 |
|---|---|---|
| 9.0 | [提示词注入](content/zh/ai-eng/part-9-security-and-governance/00-prompt-injection.md) | 为什么提示词注入没有通用解法，直接注入与间接注入差在哪，以及你上线时要带着的那套纵深防御姿态。 |
| 9.1 | [Agent 安全与致命三要素](content/zh/ai-eng/part-9-security-and-governance/01-agent-security-and-the-lethal-trifecta.md) | agent 数据外泄的三条腿威胁模型：三条腿同时凑齐为什么就是终局，以及上线前怎么至少砍掉一条。 |
| 9.2 | [护栏](content/zh/ai-eng/part-9-security-and-governance/02-guardrails.md) | 输入过滤器、输出校验器、内容审核模型，以及那个没人提醒你的流式输出与校验之争。 |
| 9.3 | [防滥用](content/zh/ai-eng/part-9-security-and-governance/03-abuse-prevention.md) | 限流、支出上限、按租户的预算，以及在账单上体现出来之前就抓住自动化滥用的那些行为信号。 |
| 9.4 | [PII 与隐私](content/zh/ai-eng/part-9-security-and-governance/04-pii-and-privacy.md) | 在调用模型之前完成检测与脱敏，理清控制者—处理者—次级处理者这条链，并选对数据保留与数据驻留的开关。 |
| 9.5 | [合规与审计](content/zh/ai-eng/part-9-security-and-governance/05-compliance-and-audit.md) | 面对 SOC 2 和欧盟 AI 法案，工程师真正要建的东西：结构化审计留痕、防篡改可验证的保留机制，以及监管方会去找的模型文档。 |
| 9.6 | [红队测试](content/zh/ai-eng/part-9-security-and-governance/06-red-teaming.md) | 自动化探测套件、人工测试流程，以及 CI 中的持续红队测试：在被别人攻破之前，先自己攻一遍。 |

</details>

<details>
<summary><a id="ai-eng-part-10"></a><strong>第 10 部分 · 生产环境中的 AI</strong></summary>

**级别：** 中级 &nbsp;·&nbsp; **适合：** 负责一个 AI 功能的账单、延迟和 on-call 的工程师。

在生产中运营 AI：托管与自托管、定制化菜单、成本与延迟工程、由评估把关的发布、SLO、事故响应，以及 AI 基础设施模式。

| # | 章节 | 内容简介 |
|---|---|---|
| 10.0 | [托管还是自托管](content/zh/ai-eng/part-10-deployment-and-cost/00-hosted-vs-self-hosted.md) | 决定自托管何时胜过按 token 计费的四行式 TCO 模型、为什么 GPU 利用率是无声的乘数，以及 2026 年的默认做法：先托管，再迁移热路径。 |
| 10.1 | [自托管要点](content/zh/ai-eng/part-10-deployment-and-cost/01-self-hosting-essentials.md) | 要把开放权重模型跑得像样，你真正需要知道的东西：vLLM 作为默认服务端、作为资源约束的 KV cache、连续批处理，以及在掏钱之前如何估算 GPU 规模。 |
| 10.2 | [定制化菜单](content/zh/ai-eng/part-10-deployment-and-cost/02-the-customization-menu.md) | 把微调、LoRA 和蒸馏当作决策而不是技术：什么时候该选哪一个、代价是什么，以及那个让坏系统看起来健康的污染陷阱。 |
| 10.3 | [成本工程](content/zh/ai-eng/part-10-deployment-and-cost/03-cost-engineering.md) | 五个相互正交的杠杆（缓存、批处理、路由、量化、蒸馏）以乘性方式叠加，在不改变模型质量的前提下，把五位数的月账单砍掉约 80%。 |
| 10.4 | [延迟工程](content/zh/ai-eng/part-10-deployment-and-cost/04-latency-engineering.md) | 为什么 LLM 延迟不是一个单一数字：五个阶段的瀑布分解、TTFT 与总耗时的区别、并行工具调用、前缀缓存，以及在 UX 侧弥合差距的感知延迟技巧。 |
| 10.5 | [变更上线](content/zh/ai-eng/part-10-deployment-and-cost/05-shipping-changes.md) | 提示词与模型变更都是部署。以评测为门禁的晋级、影子流量、金丝雀发布、即时回滚，这套纪律把随手改个提示词变成安全生产变更。 |
| 10.6 | [SLO 与事故响应](content/zh/ai-eng/part-10-deployment-and-cost/06-slos-and-incident-response.md) | AI 故障会带着 HTTP 200 返回垃圾内容。四类 AI 事故（质量回归、注入利用、成本失控、大规模幻觉）、检测它们的语义 SLI，以及把事故关进笼子的应急手册。 |
| 10.7 | [AI 基础设施模式](content/zh/ai-eng/part-10-deployment-and-cost/07-ai-infrastructure-patterns.md) | 四类 AI 特有的基础设施组件：生产环境里的向量存储、语义缓存、agent 长时任务的持久队列，以及嵌入流水线。通用部分一律交叉链接到 HLD。 |

</details>

<details>
<summary><a id="ai-eng-part-11"></a><strong>第 11 部分 · AI 产品工程</strong></summary>

**级别：** 中级到进阶 &nbsp;·&nbsp; **适合：** 决定一个 AI 功能该做什么、以及它到底有没有起作用的工程师和产品经理。

AI UX、失败恢复、准确率之外的产品指标、实验、成本建模与定价，以及会复利增长的反馈闭环。

| # | 章节 | 内容简介 |
|---|---|---|
| 11.0 | [AI 产品的用户体验（UX）](content/zh/ai-eng/part-11-ai-product/00-ai-ux.md) | 流式输出是信任信号，不是传输细节。三种故障模式（空白屏幕、中途卡住、无法核实的论断），以及各自该怎么做。 |
| 11.1 | [失败恢复的用户体验](content/zh/ai-eng/part-11-ai-product/01-failure-recovery-ux.md) | AI 的失败不是边角情况。设计好回退级联、重试入口和人工交接，让那些你拦不住的失败不至于毁掉信任。 |
| 11.2 | [AI 产品指标](content/zh/ai-eng/part-11-ai-product/02-ai-product-metrics.md) | 超越准确率。任务成功率、分流与承接与真正解决的区别、把升级当作质量信号、完成时长，以及 AI 功能留存。 |
| 11.3 | [实验](content/zh/ai-eng/part-11-ai-product/03-experimentation.md) | 当方差比确定性功能高 2-5 倍、处理组会在实验中途漂移，而读出来没有显著效果通常意味着功效不足时，怎么给 AI 功能做 A/B 测试。 |
| 11.4 | [成本建模与定价](content/zh/ai-eng/part-11-ai-product/04-cost-modeling-and-pricing.md) | 为什么边际成本重新变得重要。按任务（而不是按 token）算成本、四种定价架构，以及判断一个功能是否贵到不该上线的算术。 |
| 11.5 | [反馈与增长回路](content/zh/ai-eng/part-11-ai-product/05-feedback-and-growth-loops.md) | 大多数团队只有一份日志文件，而不是一个飞轮。一条能跑起来的数据回路需要三个前提，而真正持久的优势来自回路本身，而非数据。 |

</details>

---

## 学习计划

你并不需要计划。随便打开一章开始读就行。当你有截止日期，或者有一块明确的短板要补，这些路线才有用。

| 计划 | 适合 | 长度 |
|---|---|---|
| [面试准备：SDE1 到 SDE2](#interview-prep-sde1-to-sde2) | 通过中级系统设计面试 | 6 周 |
| [深入：SDE2 到 Senior](#deep-dive-sde2-to-senior) | 设计与主导跨团队架构 | 3 个月 |
| [AI Engineering 路线](#ai-engineering-track) | 在模型 API 上做 AI 功能 | 6 周 |
| [AI 系统设计路线](#ai-systems-design-track) | 设计规模化的 AI 系统 | 4 周 |
| [面试突击](#interview-triage) | 下周一就要面系统设计 | 1 个周末 |
| [完整课程](#the-full-curriculum) | Staff+ 级别的准备 | 数月 |

<details>
<summary><a id="interview-prep-sde1-to-sde2"></a><strong>面试准备：SDE1 到 SDE2</strong> &nbsp;—&nbsp; 6 周</summary>

**目标：** 通过一场 45–60 分钟的中高级系统设计面试。按每周 8–10 小时阅读来安排。

| 周 | 重点 | 入口 |
|---|---|---|
| 1 | 基础 | HLD [第 0 部分](#hld-part-0) 与 [第 1 部分](#hld-part-1) |
| 2 | 构建模块 I | HLD [第 2 部分](#hld-part-2) 前半：负载均衡、代理、CDN、缓存、SQL、NoSQL、分片、复制 |
| 3 | 构建模块 II | HLD 第 2 部分后半：队列、发布订阅、实时通信、限流、服务网格、对象存储、地理索引、边缘计算 |
| 4 | 核心案例 | [核心基础构件](#核心构件)的案例：URL 短链接、限流器、聊天、信息流、网页爬虫、自动补全 |
| 5 | 你的目标公司 | 更多[案例](#hld-part-8)，结合[各家公司的面试风格差异（Amazon、Google、Meta、Netflix）](content/zh/hld/part-11-interview-framework/04-company-specific-flavors.md) 来挑 |
| 6 | 面试机制 | HLD [第 11 部分](#hld-part-11) 与[权衡库](#权衡库) |

也要考编码轮？同时读 [DSA 第 0 到 9 部分](#dsa-各部分速览)。

</details>

<details>
<summary><a id="deep-dive-sde2-to-senior"></a><strong>深入：SDE2 到 Senior</strong> &nbsp;—&nbsp; 3 个月</summary>

**目标：** 达到 Senior 水平的设计能力，独立负责跨团队架构，并通过 Senior+ 的面试轮。

| 阶段 | 重点 | 入口 | 时间 |
|---|---|---|---|
| 1 | 基础与构建模块 | HLD [第 0](#hld-part-0)、[1](#hld-part-1)、[2](#hld-part-2) 部分 | 3 周 |
| 2 | 分布式理论、数据、架构 | HLD [第 3](#hld-part-3)、[4](#hld-part-4)、[5](#hld-part-5) 部分 | 4 周 |
| 3 | 把案例研究全过一遍 | HLD [第 8 部分](#hld-part-8) | 4 周 |
| 4 | 可靠性、安全、AI、面试 | HLD [第 6](#hld-part-6)、[7](#hld-part-7)、[9](#hld-part-9)、[10](#hld-part-10)、[11](#hld-part-11) 部分，以及[权衡库](#权衡库) | 2 周 |

</details>

<details>
<summary><a id="ai-engineering-track"></a><strong>AI Engineering 学习路线</strong> &nbsp;—&nbsp; 6 周</summary>

**目标：** 交付可靠、可衡量、安全且成本可控的 AI 功能。按每周 4–5 小时阅读来安排，另外留出跑代码片段的时间。

| 周 | 重点 | 入口 |
|---|---|---|
| 1 | 心智模型与模型行为 | [第 0 部分](#ai-eng-part-0) 与 [第 1 部分](#ai-eng-part-1) |
| 2 | API 与提示词 | [第 2 部分](#ai-eng-part-2) 与 [第 3 部分](#ai-eng-part-3) |
| 3 | 评测与上下文 | [第 4 部分](#ai-eng-part-4) 与 [第 5 部分](#ai-eng-part-5) |
| 4 | 检索 | [第 6 部分](#ai-eng-part-6) |
| 5 | 智能体 | [第 7 部分](#ai-eng-part-7) |
| 6 | 在生产环境跑起来 | [第 8](#ai-eng-part-8)、[9](#ai-eng-part-9)、[10](#ai-eng-part-10)、[11](#ai-eng-part-11) 部分。先扫一遍，等问题真的出现再回头细读。 |

</details>

<details>
<summary><a id="ai-systems-design-track"></a><strong>AI 系统设计路线</strong> &nbsp;—&nbsp; 4 周</summary>

**目标：** 在白板上设计 AI 系统。假定你已掌握 HLD 的基础内容。

| 周 | 重点 | 入口 |
|---|---|---|
| 1 | LLM 服务与 RAG | HLD [第 9 部分](#hld-part-9) 的前几章：LLM 服务、RAG 流水线、向量检索 |
| 2 | 智能体与评测 | HLD 第 9 部分：智能体架构、多智能体编排、LLM 评测、LLMOps |
| 3 | AI 案例研究 | [AI 系统](#ai-系统)的案例：ChatGPT、企业 RAG、编码智能体、AI 搜索、语音助手、内容审核、语义缓存、模型路由 |
| 4 | ML 基础 | HLD 第 9 部分其余内容，以及[设计一个推荐系统（Netflix / YouTube / TikTok）](content/zh/hld/part-8-case-studies/17-recommendation-system.md)案例 |

如果你还要亲手把系统建起来，就把它和 [AI Engineering 学习路线](#ai-engineering-track)搭配着读。

</details>

<details>
<summary><a id="interview-triage"></a><strong>面试急救</strong> &nbsp;—&nbsp; 一个周末</summary>

周一就有一场系统设计面试：

| 什么时候 | 入口 |
|---|---|
| 周六上午 | [如何拆解一道系统设计题](content/zh/hld/part-1-core-fundamentals/05-how-to-approach-design-questions.md)、[量级估算](content/zh/hld/part-1-core-fundamentals/04-back-of-envelope-estimation.md)、[需求界定：功能性需求、非功能性需求与 MoSCoW](content/zh/hld/part-11-interview-framework/01-requirements-scoping.md)、[系统设计面试中的画图能力](content/zh/hld/part-11-interview-framework/02-diagramming-skills.md) |
| 周六下午 | 在你熟悉的领域里看三个[案例](#hld-part-8) |
| 周日上午 | 再看三个案例，以及[权衡表达：把“看情况”说清楚](content/zh/hld/part-11-interview-framework/03-trade-off-articulation.md) |
| 周日晚上 | [各家公司的面试风格差异（Amazon、Google、Meta、Netflix）](content/zh/hld/part-11-interview-framework/04-company-specific-flavors.md)，挑你目标公司那篇 |

</details>

<details>
<summary><a id="the-full-curriculum"></a><strong>完整课程</strong> &nbsp;—&nbsp; Staff+ 准备</summary>

每本书都按顺序从头读到尾。一天一章是可持续的节奏，还能留出重读难点章节的余地。HLD 的章节末尾有练习和记忆卡片，可以用它们做主动回忆。

</details>

---

## 书是怎么写出来的

**写作规则。** 每本书都遵循[风格指南](STYLE_GUIDE.md)：

- 讲推理过程，不只给结论。读者读完应该知道该怎么判断。
- 给出推荐，并说明理由。说“看情况”，就得讲清看的是什么情况。
- 要具体。一个有出处的实测数字，胜过一句“非常快”。
- 对难度诚实。难在哪里、为什么难，都写出来。
- 写得平实：句子短，一段只讲一件事，术语第一次出现就解释清楚。

**自动检查。** 每个 pull request 都会在 CI 里跑这些检查（[`content-ci.yml`](.github/workflows/content-ci.yml)）：

| 检查项 | 能抓住什么 |
|---|---|
| [markdownlint](https://github.com/DavidAnson/markdownlint) | Markdown 结构与风格问题 |
| [typos](https://github.com/crate-ci/typos) | 拼写错误，带技术术语白名单 |
| 引用完整性 | 被引用却从未定义、或定义了却从未引用的脚注 |
| Frontmatter 校验 | 每本书的章节元数据缺失或格式不对 |
| Mermaid 校验 | 渲染会失败的图表 |
| [Vale](https://vale.sh/) | 被动语态、含糊其辞与禁用词 |
| [lychee](https://github.com/lycheeverse/lychee) | 死链，保证引用一直点得开 |

**人工评审。** 除了这些检查，评审者还会看术语是否统一、难度是否匹配本章的读者、图表是否都有说明文字、是否可无障碍阅读、引用是否指向一手来源。

---

## 参与贡献

改动多大都欢迎，从改一个错字到写完一整章。完整流程见 [CONTRIBUTING.md](CONTRIBUTING.md)。

### 可以怎么帮

| 时间 | 你能做什么 | 要先开 issue 吗 |
|---|---|---|
| 5 分钟 | 改个错别字或修个死链 | 不用 |
| 15 分钟 | 补一条缺失的引用，或把过时的数字连来源一起更新 | 不用 |
| 30 分钟 | 把一段难懂的段落讲清楚，或补一个真实案例 | 不用 |
| 一小时 | 给已有章节加一张图 | 可选 |
| 几小时 | 从技术准确性上通审一章 | 要 |
| 一个周末 | 写一章新的 | **要** |
| 长期 | 把章节翻译成你的语言 | 要 |

### 怎么贡献

1. **先读[风格指南](STYLE_GUIDE.md)。** 它讲语气、结构、图表和引用，也讲每本书的章节各有什么不同。
2. **先开 issue**：凡是半小时以上的活都先开一个，免得两个人做同一件事。用[issue 模板](https://github.com/handbook-academy/engineering-handbook/issues/new/choose)。
3. **Fork、开分支、动手改。** 分支名要能说明在做什么，比如 `fix/raft-quorum-math` 或 `add/mcp-security-chapter`。
4. **跑检查**：`npm run check:all`；也可以让 CI 在你的 pull request 上跑。
5. **提 pull request**，用 [PR 模板](.github/PULL_REQUEST_TEMPLATE.md)。小修小补一句话说明就够；新章节要讲清你的思路，并列出资料来源。
6. **回应评审。** 维护者会审这个 pull request，可能要求你修改。

### 什么样的贡献算好贡献

- **准确。** 每个数字、每条声称的性质，都要给一手来源。
- **清楚。** 按本章 frontmatter 标的难度来写。
- **有论证。** 如果你认为某章该推荐别的方案，就拿来源把理由讲出来。
- **结构对路。** 和你所改那本书的章节骨架保持一致。

**招募评审者。** 如果你实际运营过案例里涉及的系统，比如支付、聊天、信息流或视频，你的评审比一个人埋头研究一个月更有价值。开个 issue 说明你的专长领域。

---

## 仓库结构

```text
engineering-handbook/
├── content/                      The three books (CC BY-SA 4.0)
│   ├── hld/                      HLD Handbook
│   │   ├── part-0-prerequisites/ … part-11-interview-framework/
│   │   └── trade-offs/           Trade-offs Library
│   ├── dsa/                      DSA Handbook
│   │   ├── part-0-foundations/ … part-14-interview-framework/
│   │   │   └── <chapter>/lc-NNN/ sol.py, and for most problems sol.java · sol.cpp · sol.go
│   │   ├── patterns/             Pattern decision pages
│   │   ├── editorials/           Long-form editorials
│   │   ├── widgets/              Interactive widget specs (YAML)
│   │   ├── _problem-registry.yml Every LeetCode problem the book uses
│   │   └── _widget-registry.yml  Every widget
│   └── ai-eng/                   AI Engineering Handbook
│       ├── README.md             How to read the book
│       └── part-0-start-here/ … part-11-ai-product/
│           └── images/           Chapter figures
├── writing-guides/               Templates for case studies and trade-off pages
├── scripts/                      The content checks CI runs
├── .github/                      CI workflows, issue and PR templates, CODEOWNERS
├── STYLE_GUIDE.md                How we write
├── CONTRIBUTING.md               How to contribute
├── CODE_OF_CONDUCT.md            Contributor Covenant
├── SECURITY.md                   How to report a security issue
├── CITATION.cff                  Citation metadata
└── LICENSE                       CC BY-SA 4.0
```

---

## 开发环境

贡献不需要装任何东西。改 Markdown、提 pull request，CI 会做检查。想先在本地跑检查的话：

**你需要** [Node.js 20+](https://nodejs.org/)（版本由 `.nvmrc` 固定）和 npm。[typos](https://github.com/crate-ci/typos)、[lychee](https://github.com/lycheeverse/lychee) 和 [Vale](https://vale.sh/) 是可选的。

```bash
git clone https://github.com/handbook-academy/engineering-handbook.git
cd engineering-handbook
nvm use        # the Node version from .nvmrc
npm install    # markdownlint, Mermaid and the other check tools
```

```bash
npm run check:all          # everything CI runs

npm run lint               # Markdown style
npm run check:frontmatter  # chapter metadata
npm run check:citations    # footnotes are defined and used
npm run check:mermaid      # diagrams parse
npm run check:typos        # spelling
npm run check:links        # dead links (slow)
npm run prose              # Vale prose style (needs Vale)
```

**在浏览器里编辑。** [Markdownlab](https://markdownlab.vercel.app)（[源码](https://github.com/invincible04/markdownlab)）是一个带实时预览的 Markdown 编辑器，能渲染 Mermaid，也支持这几本书用的 callout 和脚注。在任意章节的 GitHub URL 前面加上 `https://markdownlab.vercel.app/`，就能在那里打开它：

```text
https://markdownlab.vercel.app/https://github.com/handbook-academy/engineering-handbook/blob/main/content/hld/part-1-core-fundamentals/00-scalability.md
```

仓库里还带了 `.editorconfig`、`.markdownlint.json`、`.vale.ini` 和 `.typos.toml`，多数编辑器会自动读到这些项目设置。

---

## 常见问题

<details>
<summary><strong>真的免费吗？</strong></summary>

免费。本仓库的每一章，任何人都能随时免费阅读，GitHub 上和 [handbook.academy](https://handbook.academy) 上都是，不需要登录。

这一点由许可证保证。这里的一切都是 CC BY-SA 4.0，任何人都可以复制、分享和改编，只要改编后的作品沿用同一许可证。谁都没法把它收回或变成独占的，包括我们。

</details>

<details>
<summary><strong>决策是怎么做出来的？</strong></summary>

项目由创始人和维护者 [@invincible04](https://github.com/invincible04) 主导：书写什么内容由他定，每一处改动都由他最后拍板。

书的调研、起草和评审都用上了目前最强的 AI 模型。重活交给它们：收集来源、起草和反复改写章节、拿引用的来源核对论断、检查准确性和清晰度。但最终决定权不在它们手上。什么能发由 @invincible04 决定；来源撑不住的内容，要么重写，要么删掉。

各类改动分别怎么定：

- **错字、死链和事实性更正：** 任何维护者都可以合并。
- **新章节或大幅改写：** 需要开 issue 说明想法，并由维护者批准。
- **改动课程结构（新增、改名或重排某个部分）：** 需要开一个 Discussion，至少开放一周才能合并。
- **许可证：** CC BY-SA 4.0 是长期承诺，要改它得所有贡献者同意。

连续几个月贡献稳定的人，可能会被邀请成为维护者，在自己负责的范围内有合并权限，并对课程结构决策有投票权。[CODEOWNERS](.github/CODEOWNERS) 写明了谁审哪一块。

</details>

<details>
<summary><strong>我能拿它做收费课程吗？</strong></summary>

可以。CC BY-SA 4.0 允许商用，只要做到：

1. 注明出处：“Adapted from The Engineering Handbook, CC BY-SA 4.0, https://github.com/handbook-academy/engineering-handbook”。
2. 你的版本也按同样的 CC BY-SA 4.0 条款授权。
3. 说明你改了什么。

你可以基于这些内容讲课、办训练营、做视频或出书。但不能去掉署名，也不能把正文改成专有许可。

</details>

<details>
<summary><strong>我能翻译它吗？</strong></summary>

非常欢迎。开个 issue 说明要译成哪种语言、从哪几章开始。译文可以放在本仓库的平行目录里，比如印地语用 `content-hi/`，也可以放在关联的姊妹仓库。

</details>

<details>
<summary><strong>HLD 这本书和 system-design-primer 或 Alex Xu 的书有什么不同？</strong></summary>

[system-design-primer](https://github.com/donnemartin/system-design-primer) 是一份很好的索引，指向别人的文章和演讲。HLD Handbook 则把页面上的每个主题都讲透，所以两者正好互补：primer 给资源上的广度，这本书给深度。

Alex Xu 的 *System Design Interview* 系列很出色，这里的案例研究经常引用它们。HLD Handbook 补上了案例背后的基础章节，覆盖更新的主题（LLM、RAG、agent、CRDT、后量子密码、MCP），免费，而且接受 pull request。

</details>

<details>
<summary><strong>DSA 这本书和 NeetCode、LeetCode 题解或《Cracking the Coding Interview》有什么不同？</strong></summary>

- **按模式组织，不按题目组织。** 每章讲一个数据结构或模式，LeetCode 题目是其中的例题。
- **四种语言对照。** 先用 Python 读章节，再打开 Java、C++ 或 Go 解法，看你面试要用的语言怎么写。
- **决策页。** 两种解法都合适时，模式决策页告诉你先抓哪个。
- **逐步动画。** 重点章节配有组件规范，把算法一步步演示出来。
- **开放。** 免费，并且接受 pull request。

</details>

<details>
<summary><strong>AI Engineering 这本书和 HLD 第 9 部分、和别的 AI 书是什么关系？</strong></summary>

HLD 第 9 部分回答的是“这个 AI 系统在规模上该怎么设计”。AI Engineering Handbook 回答的是“我该怎么把它建起来、跑起来”：prompt、评估集、检索管线、agent 循环、调用链、账单。两本书互相链接。

Chip Huyen 的 *AI Engineering* 是这个领域最完整的参考书，本书也引用它。AI Engineering Handbook 免费，接受 pull request，所以过时的价格、改了名的 API 参数都能很快修掉；它会给出默认选择（先上 workflow 再上 agent，先用 pgvector 再上专用向量数据库），并且先讲评估，再讲 RAG 和 agent。

</details>

<details>
<summary><strong>GitHub 上的内容和网站上的内容一样吗？</strong></summary>

一样。网站上发布的就是本仓库里的同一份文字。

</details>

<details>
<summary><strong>为什么没有一份外部资源的“awesome list”？</strong></summary>

来源都在该出现的地方就地引用。单列一份清单等于把同样的活再做一遍，而且过时得更快。一个资源要是值得读，章节里就会引它。

</details>

<details>
<summary><strong>我想写一章新主题，怎么做？</strong></summary>

用[写作模板](https://github.com/handbook-academy/engineering-handbook/issues/new/choose)开个 issue，写清：

1. 这一章属于哪本书、哪个部分。
2. 为什么该有这一章：缺了什么，或者新在哪里。
3. 一两段的大纲。
4. 你的一手来源。

维护者会回复，就范围给出意见。通过之后，fork、起草、提 pull request。案例研究用 [`writing-guides/case-study-template.md`](writing-guides/case-study-template.md)，权衡页用 [`writing-guides/trade-off-template.md`](writing-guides/trade-off-template.md)，其他章节都按[风格指南](STYLE_GUIDE.md)来。

</details>

---

## 许可证

本仓库的一切（三本书、写作指南、风格指南、检查脚本、模板和配置）都按 **[Creative Commons Attribution-ShareAlike 4.0 International](LICENSE)** 授权。

| 你可以 | 你必须 |
|---|---|
| 永久地阅读、分享、打印、截图 | **署名**原作者，并链接回本仓库 |
| 基于它做课程、训练营、视频或书 | 以**同样的 CC BY-SA 4.0 许可**发布你的版本 |
| 把它翻译成任何语言 | 不能改成专有许可，也不能加额外限制 |
| 用于商业用途 |  |

正因为谁都能复制和分享，所以谁也没法把文字收回或变成独占的。每一章都对所有人免费开放，随时可读，GitHub 上和 **[handbook.academy](https://handbook.academy)** 上都是，不需要登录。

---

## 引用

在论文、文章、书或课程里提到这个项目时，可以直接用这个格式：

```bibtex
@misc{engineering-handbook,
  author       = {Soni, Aayush},
  title        = {The Engineering Handbook: Open-Source Engineering Curricula (HLD + DSA + AI Engineering)},
  year         = {2026},
  publisher    = {GitHub},
  url          = {https://github.com/handbook-academy/engineering-handbook},
  note         = {CC BY-SA 4.0}
}
```

> 直接可用的引用格式：*"The Engineering Handbook (HLD + DSA + AI Engineering)" by Aayush Soni, https://github.com/handbook-academy/engineering-handbook, CC BY-SA 4.0.*

GitHub 的 **Cite this repository** 按钮、Zenodo、Zotero 以及其他文献管理工具都能直接读 [`CITATION.cff`](CITATION.cff)。

---

## 致谢

这几本书站在许多人的工作之上，各章随处都在引用他们。影响最大的这些：

**系统设计**

- **Martin Kleppmann：** [*Designing Data-Intensive Applications*](https://dataintensive.net/) 是分布式系统和数据两部分的地基。
- **Alex Xu：** *System Design Interview* 系列确立了面试案例的写法，HLD 的案例就建立在这个格式上。
- **Chip Huyen：** [*Designing Machine Learning Systems*](https://www.oreilly.com/library/view/designing-machine-learning/9781098107956/) 塑造了 HLD 第 9 部分。
- **Betsy Beyer 等（Google）：** [*Site Reliability Engineering*](https://sre.google/sre-book/table-of-contents/) 和 [*The Site Reliability Workbook*](https://sre.google/workbook/table-of-contents/) 是可靠性部分的底本。
- **Mark Richards 和 Neal Ford：** [*Fundamentals of Software Architecture*](https://www.oreilly.com/library/view/fundamentals-of-software/9781492043447/) 和 [*Software Architecture: The Hard Parts*](https://www.oreilly.com/library/view/software-architecture-the/9781492086888/) 启发了架构模式的内容。

**数据结构与算法**

- **Cormen、Leiserson、Rivest 和 Stein：** [*Introduction to Algorithms*](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/) 提供了证明和复杂度论证的底子。
- **Robert Sedgewick 和 Kevin Wayne：** [*Algorithms*](https://algs4.cs.princeton.edu/home/) 影响了数据结构的讲法。
- **Jon Bentley：** *Programming Pearls*，提示卡的灵感来源。
- **Steven 和 Felix Halim：** [*Competitive Programming*](https://cpbook.net/)，竞赛技巧取自这里。
- **Aziz、Lee 和 Prakash：** [*Elements of Programming Interviews*](https://elementsofprogramminginterviews.com/)，以及 **Gayle Laakmann McDowell：** [*Cracking the Coding Interview*](https://www.crackingthecodinginterview.com/)，为面试讲解立了标准。
- **[NeetCode](https://neetcode.io/)** 和 **LeetCode 社区**，练习梯度和题集来自他们。

**AI 工程**

- **Chip Huyen：** [*AI Engineering*](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) 划出了 AI 工程与 ML 工程的界线。
- **Shawn Wang（swyx）：** [“The Rise of the AI Engineer”](https://www.latent.space/p/ai-engineer) 给这个角色命了名，也画出了全书据以组织的 API 界线。
- **Hamel Husain 和 Shreya Shankar：** 他们关于评估和错误分析的文章塑造了评估部分。
- **Simon Willison：** 他在 prompt injection 和“致命三要素”上的工作，是安全部分的基础。
- **Anthropic：** [“Building effective agents”](https://www.anthropic.com/engineering/building-effective-agents) 是 workflow 与 agent 这条分界的出处。

**还有更多人**

- Donne Martin 的 **[system-design-primer](https://github.com/donnemartin/system-design-primer)**，以及 Kubernetes、CNCF、OpenTelemetry 社区和各个 Apache 项目。
- SIGMOD、VLDB、OSDI、SOSP、NSDI 论文的作者，以及 IETF、W3C、IANA 的标准。
- **每一位写过公开事后复盘的工程师。** 关于优雅降级、爆炸半径和测试的边界，真实故障教给我们的最多。

这些思想我们都用自己的话讲，配自己画的图，并注明原始出处。

---

## 社区

- **[GitHub Discussions](https://github.com/handbook-academy/engineering-handbook/discussions)：** 提问、反馈和想法
- **[GitHub Issues](https://github.com/handbook-academy/engineering-handbook/issues)：** 纠错、章节需求和提案
- **发邮件到 [hello@handbook.academy](mailto:hello@handbook.academy)：** 私密或敏感事项，包括行为准则和许可问题
- **[安全公告](https://github.com/handbook-academy/engineering-handbook/security/advisories/new)：** 脚本或 CI 里的安全问题（[政策](SECURITY.md)）
- **维护者：** [@invincible04](https://github.com/invincible04)

如果这几本书对你有用，给仓库点个 star。star 能帮新读者和贡献者找到这个项目。

[![Star History Chart](https://api.star-history.com/svg?repos=handbook-academy/engineering-handbook&type=Date)](https://star-history.com/#handbook-academy/engineering-handbook&Date)

---

<div align="center">

### 开始阅读

**HLD：** [网络基础](content/zh/hld/part-0-prerequisites/00-networking-fundamentals.md) · [可扩展性](content/zh/hld/part-1-core-fundamentals/00-scalability.md) · [URL 短链接服务](content/zh/hld/part-8-case-studies/00-url-shortener.md) · [权衡库](#权衡库)

**DSA：** [数组](content/zh/dsa/part-1-linear-data-structures/00-arrays.md) · [双指针](content/zh/dsa/part-3-pointers-window-prefix/00-two-pointers-opposite.md) · [滑动窗口](content/zh/dsa/part-3-pointers-window-prefix/03-sliding-window-variable.md) · [模式决策页](#模式决策页)

**AI Engineering：** [AI 工程师是什么](content/zh/ai-eng/part-0-start-here/00-what-an-ai-engineer-is.md) · [升级阶梯](content/zh/ai-eng/part-0-start-here/03-the-escalation-ladder.md) · [你的第一套评估集](content/zh/ai-eng/part-4-eval-fundamentals/02-your-first-eval-set.md) · [Workflow 与 agent](content/zh/ai-eng/part-7-agent-engineering/00-workflow-vs-agent.md)

许可 [CC BY-SA 4.0](LICENSE) · [handbook.academy](https://handbook.academy)

</div>
