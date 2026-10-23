# 写作规范

本规范规定在 The Engineering Handbook（HLD、DSA、AI Engineering，以及本仓库未来任何课程）里**我们怎么写**。每一章都会在 CI 和人工评审中按这些规则检查。

如果你是贡献者，先通读一遍，写作时把它开在一个标签页里。

## 目录

- [核心原则](#核心原则)
- [口吻与语气](#口吻与语气)
- [结构](#结构)
- [权衡表](#权衡表)
- [格式规范](#格式规范)
- [Frontmatter 结构](#frontmatter-结构)
- [DSA 专属差异](#dsa-专属差异)
- [AI Engineering 专属差异](#ai-engineering-专属差异)
- [图表](#图表)
- [代码块](#代码块)
- [链接与引用](#链接与引用)
- [数字与单位](#数字与单位)
- [标点与排版](#标点与排版)
- [无障碍](#无障碍)
- [反模式](#反模式)

## 核心原则

1. **教推理，不教结论。** 读者读完应该知道*怎么判断*，而不只是答案是什么。
2. **默认给立场。** 挑一个推荐方案并说明理由。“看情况”是逃避，除非你说清楚看什么情况。
3. **具体压过抽象。**“单节点 Redis 能扛 100K ops/sec”胜过“Redis 很快”。
4. **坦白复杂度。** 说“这确实很难，原因如下”，而不是“照 X 做就行”。
5. **正文保持在初中生能读懂的水平。** 句子要短，一段一个意思，术语首次出现就解释。但不要把技术内容注水。
6. **讲主题，不讲文档。** 正文解释的是主题本身。不要宣布某张表是怎么组织的、不要提示略读的读者可能漏看哪一行、不要在段落上盖日期戳，也不要叙述你自己的编辑决定。如果某个结构上的点必须说，就说一次，而且要放在推进理解的正文里。元评论不是严谨。

## 口吻与语气

### 主动语态

> 负载均衡器分发请求。

不要这样写：

> 请求由负载均衡器分发。

### 第二人称

> 你会注意到，尾部延迟在 p95 附近开始偏离均值。

不要这样写：

> 有人可能会观察到，尾部延迟……

### 有观点

> 需要有序的分区流时用 Kafka；不需要就用 SQS。

不要这样写：

> Kafka 和 SQS 都是消息队列的可选方案。

### 坦率谈权衡

> 这个方案在 100K QPS 上下就会垮，因为每个请求都要拿一把分布式锁，锁争用成了瓶颈。

不要这样写：

> 这个方案扩展性很好。

## 结构

每一篇教学章节（Part 0 到 7，以及 9、10、11）都遵循这套骨架：

```text
1. Frontmatter
2. # 标题（H1）
3. > TL;DR 引用块
4. ## 学习目标（清单）
5. ## 直觉理解
6. ## 原理（含 ### 子小节）
7. ## 真实案例
8. ## 权衡（表格）                  # 条件性；见下文“权衡表”
9. ## 常见陷阱（含 callout）
10. ## 练习（含 <details> 提示与答案）
11. ## 要点回顾
12. ## 延伸阅读
13. ## 记忆卡片
```

案例章节（Part 8）遵循 10 步模板。标准范例见 [`content/hld/part-8-case-studies/00-url-shortener.md`](content/hld/part-8-case-studies/00-url-shortener.md)：

```text
1. 问题陈述
2. 澄清式提问
3. 功能与非功能需求
4. 容量估算
5. API 设计
6. 数据模型
7. 高层架构
8. 深入探讨（选 3 个）
9. 权衡表                          # 条件性；见下文“权衡表”
10. 扩展与瓶颈
+ 追问
+ 常见陷阱
+ 失效模式
+ 要点回顾
+ 延伸阅读
+ 记忆卡片
```

## 权衡表

只有当一章为同一个设计决策给出**三种或更多可互换方案**时，权衡表才配得上它的位置。五列格式默默承诺了“行与行等价”；一旦某张表违背这个承诺，读者就会被导向错误的选择。

**规则 1：这一节是有条件的。** 只有同时满足三个条件才写 `## Trade-offs`：(1) 本章有 ≥3 个可互换方案；(2) 读者会拿这张表在行之间做选择；(3) 每一行都是站得住的选项。任一条不成立，就换个标题：

| 如果这些行是…… | 改用这个标题 |
|---|---|
| 互补的控制措施（全都要上、分层叠加） | `## Defense in depth` |
| 归到一起的独立决策（这里用二选一的表没问题） | `## Design decisions` |
| 工具或厂商，而不是架构 | `## Tool selection` |
| 选择由使用场景决定 | `## When to use which` |
| 一套方法论，或不对称的两个选项（其中一个选项需要连贯论证，塞不进单元格） | 用正文，可以配一张 Mermaid 流程图 |

只要比较是对称的，表格就是首选格式（每个选项都有“优点”“缺点”“适用场景”“我们的选择”四个单元格，而且能干净地对应上）。对称的二选一决策应该放在 `## Design decisions` 或类似标题下的两行表里，不要写成正文。只有当决策不对称（某个选项有独特的失效模式，套不进表格结构），或者内容本身是方法论而非比较时，正文才是合适的格式。

**规则 2：禁止写反模式行。** 任何单元格都不能出现“Never”“Avoid”“Almost never”“Only for legacy”“Don't use”或等价说法，也包括“Avoid on greenfield”“Almost never in practice”这类软化版本。把警告挪到 `## Common Pitfalls`，写成 `> [!WARNING]` callout，里面给出具体失效模式、一手来源引用，以及实际该用的替代方案：

```markdown
> [!WARNING]
> **最后写入者胜出会静默丢弃并发写入。** 两个副本在复制窗口内接受对同一个键的写入，
> 最终只产出一条按时间戳胜出的写入；另一条被丢弃，
> 用户看不到任何信号
> ([Shapiro et al., 2011](https://doi.org/10.1007/978-3-642-24550-3_29))。
> 协作文档、共享文件或购物车这类场景，请改用 OR-Set、RGA/Yjs 文档，
> 或显式的双副本分叉。
```

**规则 3：改那一行，不要给表加注解。** 如果某一行可能被误用，就把列改锐利，让风险在行内自己显现：把“缺点”单元格收紧到具体失效模式、把“适用场景”的条件收窄，或者让“我们的选择”更有态度。如果收窄之后风险还在，就按规则 2 把这一行挪到 `## Common Pitfalls` 的 `> [!WARNING]` callout 里。表下加说明是最后手段，只在风险确实与行结构正交时才用（比如针对某个量级数字的校准提示）。那种情况下写一句斜体，点明具体失效模式。默认不加说明。不要写“略读时这张表可能怎么读”之类的元评论，不要给表盖日期戳，也不要宣布删了什么、改了什么名——那些属于提交信息，不属于内容。

**规则 4：用默认结构，或者换成更好的。** 默认是：

```markdown
| 方案 | 优点 | 缺点 | 适用场景 | 我们的选择 |
|----------|------|------|-----------|----------|
| ...      | ...  | ...  | ...       | ...      |
```

从这里起步。只要每行的优点和缺点确实不同，默认结构就够用。一致性级别的对比见 [`content/hld/part-3-distributed-systems-theory/01-consistency-deep-dive.md`](content/hld/part-3-distributed-systems-theory/01-consistency-deep-dive.md)。

只要主题有更清楚的维度，就换成与决策相关的列：吞吐量、延迟、顺序性、成本、协调开销、写放大——凡是真的能把行区分开的东西。值在各行之间大致相同的列，砍掉；“我们的选择”这一列要留着。有三章示范了这种做法：

- [`content/hld/part-2-building-blocks/08-message-queues-streaming.md`](content/hld/part-2-building-blocks/08-message-queues-streaming.md)：各列依次为 `Throughput | Ordering | Replay | Ops burden | Cost model | Our pick`。
- [`content/hld/part-4-data-systems/00-storage-engines.md`](content/hld/part-4-data-systems/00-storage-engines.md)：数值化的写放大列和压缩比列。
- [`content/hld/part-8-case-studies/27-unique-id-generator.md`](content/hld/part-8-case-studies/27-unique-id-generator.md)：各列依次为 `Bit-width | Coordination | B-tree locality | Our pick`。

## 格式规范

### 标题

- 每个文件恰好一个 `# H1`，与 frontmatter 的 `title` 一致。
- 大节用 `## H2`。
- 小节用 `### H3`。
- 不要跳级（不能从 `##` 直接跳到 `####`）。
- 句首大写，不要每个词都大写：非模板标题写 `## Common pitfalls`，不写 `## Common Pitfalls`。（上面的模板小节名是例外。）

### Callout（GitHub 提示块）

只用 GitHub 原生的五种提示类型：

```markdown
> [!NOTE]
> 有用的背景信息。

> [!TIP]
> 来自生产经验的实用技巧。

> [!IMPORTANT]
> 读者绝不能漏掉的关键信息。

> [!WARNING]
> 常见错误或容易忽略的坑。

> [!CAUTION]
> 在生产环境里很危险的操作。
```

不要自造提示类型。不要用 emoji 当小节标记。

### 表格

一定要有表头。文本列左对齐；数值列在便于比较时右对齐。

```markdown
| 方案 | 优点 | 缺点 | 适用场景 |
|----------|------|------|-----------|
| A | ... | ... | ... |
```

超过约 10 行的表要拆分或总结。

### 列表

- 无序列表用 `-`（连字符）。同一个文件里不要混用 `*` 和 `-`。
- 有序列表每一项都写 `1.`（Markdown 会自动编号）。这样 diff 干净。
- 嵌套列表缩进两个空格。

## Frontmatter 结构

每个内容文件都必须有合法的 YAML frontmatter：

```yaml
---
title: "Caching Strategies"                # required, string
description: "Deep dive into caching..."   # required, 1-sentence, < 160 chars
part: 2                                    # required, integer 0-11
module: "2.3"                              # required, string "X.Y"
difficulty: intermediate                   # required, one of: beginner | intermediate | advanced
prerequisites:                             # required, array (can be empty)
  - 0.1-os-essentials
  - 1.1-latency-and-throughput
date_created: 2026-04-29                   # required, ISO-8601 date
date_updated: 2026-04-29                   # required, ISO-8601 date
tags: [cache, performance]                 # optional, array — see canonical tags below
technologies: [Redis, Memcached]           # optional, array — must be on the allowlist in scripts/technologies.json
---
```

前置章节的 slug 用 `X.Y-slug-with-dashes` 形式：`X.Y` 对应目标章节的 `module` 字段，`slug-with-dashes` 是不含 `.md` 扩展名的文件名。

### 规范标签

`tags:` 放的是**主题概念**，不是工具名。工具名归 `technologies:`。标签只用下面这约 40 个规范值——标签云和搜索筛选之所以有用，靠的就是这一点。

不要临时造标签（比如 `slack`、`whisper`、`bloom-filter`）。案例章节已经声明了 `theme:`，别在 `tags:` 里重复。

| 分类 | 规范标签 |
|---|---|
| **分布式系统** | `consistency`, `replication`, `partitioning`, `consensus`, `transactions`, `idempotency`, `fanout` |
| **数据存储** | `sql`, `nosql`, `cache`, `storage-engines`, `vector-search`, `time-series`, `search`, `geospatial` |
| **消息** | `queue`, `stream-processing`, `cdc` |
| **网络** | `http-api`, `real-time`, `cdn`, `dns`, `load-balancing` |
| **架构** | `microservices`, `event-driven`, `serverless`, `api-gateway`, `multi-tenant` |
| **流量** | `rate-limiting`, `identifier` |
| **可靠性** | `observability`, `resilience`, `deployment`, `slo` |
| **安全** | `authn-authz`, `encryption`, `zero-trust` |
| **AI / ML（机器学习）** | `llm`, `rag`, `ml-systems`, `ai-safety`, `agents` |
| **媒体** | `video` |
| **横切关注点** | `cost` |

### Part 8 案例主题

`content/hld/part-8-case-studies/` 下每一章都必须多声明一个字段 `theme:`，把它归进一个教学分组。主题只服务于编辑视角：它从不改变章节的 URL、它在 Part 8 里的编号，也不改变任何交叉引用。它让 Part 8 落地页和 README 能把 56 个案例当成菜单呈现，而不是排成一条队列。

| 主题 | 这里放什么 | 示例 |
|---|---|---|
| `core-primitives` | 面试经典题。一章一个标准原语，从零重建。 | url-shortener, rate-limiter, key-value-store, chat-system, distributed-cache |
| `consumer-products` | 由多个原语组合而成的终端用户产品。 | netflix, ride-hailing, food-delivery, video-conferencing |
| `ai-systems` | 以 LLM 或 ML 为核心的系统，推理、检索或 agent 行为是设计的核心挑战。 | chatgpt, enterprise-rag, coding-agent, semantic-cache, recommendation-system |
| `financial-systems` | 正确性至关重要、带监管或审计要求的系统。 | payment-system, stock-exchange, brokerage-trading, fraud-detection |
| `infrastructure-platforms` | 其他工程师在其上构建的平台：可观测性、CI/CD、API 边缘、路由。 | api-gateway, cicd-platform, observability-platform, dns-service, metrics-pipeline |

如果新案例不能干净地落进某个主题，就选最接近的那个，并在 PR 描述里说明这处含糊。不要不经讨论就新造主题；`scripts/check-frontmatter.mjs` 会强制校验白名单。

## DSA 专属差异

`content/dsa/` 下的 DSA 章节和 HLD 章节形态不同。本规范里的大部分规则（口吻、引用、Mermaid、正文卫生）原样适用；差异在章节形态、标识符约定和代码示例的布局。

**章节形态。** 练习优先：开头是一屏能看完的速查表（操作、时间、空间、坑点），然后深入讲这个数据结构或模式，接着是一串“提示卡”——每张卡是一道有代表性的 LeetCode 题，配一个自己写的实例推演、一个用 `<details>` 折叠的解法讲解，以及一个“常见错误”块。标准范例是 [`content/dsa/part-1-linear-data-structures/00-arrays.md`](content/dsa/part-1-linear-data-structures/00-arrays.md)。

**Frontmatter。** DSA 章节用 `slug`、`part`、`chapter`、`prerequisites`、`date_updated`、`languages`、`canonical_test`、`widgets` 和 `ladder`（含 `core`、`stretch`、`star` 三个键）。它们**不**用 HLD 的 `module`、`tags`、`technologies` 字段。`slug` 必须等于不含 `.md` 的文件名。`languages` 是 `[python, java, cpp, go]` 的非空子集。`canonical_test` 要么是 null，要么是 `LC-NNNN` 形式的 ID，且必须出现在 [`_problem-registry.yml`](content/dsa/_problem-registry.yml) 里。

**代码示例。** 本章讲解的每道 LeetCode 题都放在同级目录 `<chapter-slug>/lc-NNN/` 下，里面有四个解法文件（`sol.py`、`sol.java`、`sol.cpp`、`sol.go`）外加一个用于测试的 `cases.json`。章节正文里用围栏块内联 Python 解法；在单独一行“其他语言”里链接 Java/C++/Go 版本。解法必须自包含、符合语言习惯，并且能通过 `cases.json` 里的用例。

**标识符约定。** 题目用 `LC-NNN` 引用（例如 `LC-088`、`LC-189`），并在 [`content/dsa/_problem-registry.yml`](content/dsa/_problem-registry.yml) 注册。交互组件用 `w-NN-<slug>` 引用标准模式组件（例如 `w-15-lru-cache`），或用 `e-LCNNN-<slug>` 引用题解组件，并在 [`content/dsa/_widget-registry.yml`](content/dsa/_widget-registry.yml) 注册。在 GitHub 上，组件提示行链接到 YAML 规范；渲染出来的组件本身在网站上。

**题解与模式。** 逐题的长篇题解放在 `content/dsa/editorials/` 下；模式决策参考（每个主要模式家族一篇）放在 `content/dsa/patterns/` 下。两者的 frontmatter 结构都更薄——照现有文件的形态写。

## AI Engineering 专属差异

`content/ai-eng/` 下的 AI Engineering 章节沿用本规范的口吻、引用和无障碍规则。不同之处在章节形态、frontmatter 和图表。

**章节形态。** 没有固定骨架。每章按主题需要自己组织：多数从一个真实事故或一个失败案例开场，解释机制，最后落到读者能用上的东西（清单、决策规则，或一段能跑的代码）。不使用 HLD 模板里的 TL;DR、学习目标、记忆卡片等小节。每章只讲一个概念，篇幅要短；长度是挣来的，绝不注水。

**边界线。** 如果某个主题回答的是“我会在白板上画什么”，它属于 [HLD Part 9](content/hld/part-9-ai-ml-system-design/)，AI Engineering 那一章链接过去。如果回答的是“我会在编辑器里打开什么”，它属于 `content/ai-eng/`。

**Frontmatter。** AI Engineering 章节用 `title`、`description`、`slug`、`part`、`chapter`、`difficulty`、`prerequisites`、`tags`、`date_created`、`date_updated`。它们**不**用 HLD 的 `module` 或 `technologies` 字段。`slug` 必须等于不含 `.md` 的文件名；`part` 和 `chapter` 必须与目录和文件名前缀一致。`prerequisites` 用 DSA 的路径形式：跨 Part 写 `part-N-slug/NN-chapter-slug`，同 Part 内写 `NN-chapter-slug`，并且必须指向真实存在的章节。`tags` 用上面的规范标签表。

```yaml
---
title: "MCP security"
description: "The four attack classes MCP creates..."
slug: 05-mcp-security
part: 7
chapter: 5
difficulty: intermediate
prerequisites:
  - 04-mcp-model-context-protocol
  - 03-tool-design
tags: [llm, agents, ai-safety]
date_created: 2026-06-12
date_updated: 2026-06-12
---
```

**要图，但不要 Mermaid 配额。** “至少三张图”的下限在这里不适用。教学插图是极简的 PNG，存在章节旁边的 `content/ai-eng/<part-dir>/images/<NN>-<name>.png`，通常每章一到三张，只在真能讲清问题时才用。每张图都要有描述性的 alt 文本，说明图里展示了什么，紧跟一句斜体图注。Mermaid 是例外，每章最多一张，只在流程图比图片讲得更好时用。

**代码。** 程序代码用 Python，面向 3.10+ 和官方厂商 SDK。SQL、YAML、JSON、shell 块只在主题本身就是配置或查询时出现。

**易过时的事实。** 价格、模型名、上下文长度、基准分数都要带“截至”日期和带脚注的来源。优先用厂商自己的文档或定价页，而不是博客上的总结。

**警告。** 会烧钱或引发事故的生产陷阱，用 GitHub 的 `> [!WARNING]` callout。

## 图表

我们只用 **Mermaid** 和 **Excalidraw**。ASCII 方框图（用制表符画出来的 `+--+ | +--+` 那种）在教学章节和案例里**不允许**出现。它们会和同一页的 Mermaid 逐渐脱节，在比例字体下会错位，每次编辑都要手工重新对齐，而且提供不了任何 Mermaid 表达得更好的信息。如果你接手了旧章节里的一张，转成 Mermaid 或者删掉。

每一篇教学章节都需要**至少三张 Mermaid 图**：一张高层架构总览，一张展示动态行为的时序图或状态图，以及一张作者自选的图（决策树、数据模型、部署拓扑等）。案例章节（Part 8）至少需要四张。

每张图正下方都有一句斜体图注，说明要点。示例：

```markdown
\```mermaid
flowchart LR
    Client --> LB[负载均衡器]
    LB --> S1[服务器 1]
    LB --> S2[服务器 2]
\```

*负载均衡器把请求分发到一组无状态服务器上；要加容量，多起几台服务器就行。*
```

### Mermaid（默认选择）

Mermaid 在 GitHub 上原生渲染（2022 年 2 月起），在渲染站点上也通过一个懒加载的客户端 bundle 渲染。

```markdown
\```mermaid
sequenceDiagram
    participant Client as 客户端
    participant LB as 负载均衡器
    participant Server as 服务器
    Client->>LB: HTTP 请求
    LB->>Server: 转发
    Server-->>LB: 响应
    LB-->>Client: 响应
\```
```

按要表达的意思选图类型：

| 类型 | 用于 |
|------|---------|
| `flowchart LR` 或 `flowchart TD` | 高层架构、请求路径、组件布局 |
| `sequenceDiagram` | 协议流程（TCP 握手、API 往返、共识轮次） |
| `stateDiagram-v2` | 状态机（连接状态、消息生命周期、任务状态） |
| `erDiagram` | 数据模型、表关系 |
| `graph LR` | 决策树、依赖图 |
| `gantt` | 时间线（故障恢复阶段、上线阶段） |

让 Mermaid 图好读的几条建议：

- 节点用给人看的标签，不要用变量名：`LB[Load Balancer]`，不要写 `LB[LB]`。
- 用 `subgraph` 把相关组件分组（例如“数据面”“控制面”）。
- 边的含义不是自明的时候就标上：`Client -->|GET /api| LB`。
- 图控制在 15 个节点以内。超了就拆成两张，或者改用 Excalidraw。

### Excalidraw（Mermaid 表达不了的图）

当一张图确实套不进 Mermaid 的词汇表时，用 Excalidraw：多区域部署拓扑、在白板照片上做自由标注、带信任边界威胁模型，或者超过 15 个节点的密集多服务架构。

源文件和导出图都要提交：

- `diagrams/<chapter-slug>.excalidraw`（源文件，别人可以改）
- `diagrams/<chapter-slug>.svg`（渲染结果，Markdown 里引用它）

在 Markdown 里这样引用：

```markdown
![多区域部署拓扑](../../diagrams/url-shortener.svg)

*每个区域跑一套完整的栈；全局负载均衡器把用户路由到最近且健康的区域。*
```

一定要写描述性的 `alt` 文本（第一个参数）和图注（下面那行斜体）。`alt` 文本服务屏幕阅读器和搜索，图注服务读者。

## 代码块

代码块一律标语言，方便语法高亮：

```markdown
\```python
def example():
    pass
\```
```

我们常用的语言：`bash`、`python`、`javascript`、`typescript`、`go`、`rust`、`java`、`yaml`、`json`、`sql`、`http`、`nginx`、`dockerfile`、`mermaid`。

代码块控制在约 40 行以内。更长的抽到 `examples/` 下的文件里，再链接过去。

## 链接与引用

### 内部链接

用相对路径，目录写磁盘上的真实目录名，文件名带补零编号：

```markdown
见 [CAP 与 PACELC](../part-1-core-fundamentals/04-cap-and-pacelc.md)。
前置框架见 [一致性模型：读者到底看到什么](../part-1-core-fundamentals/03-consistency-models.md)。
```

这种写法在 GitHub 上浏览仓库时可用。站点构建时，生成器会把这些链接改写成公开 URL（例如 `/core-fundamentals/cap-and-pacelc/`）。如果跨章链接指向的文件不存在，构建就会失败；改掉错字或过期的引用。

**交叉引用规则（强制）：** 正文里提到另一章时，用可点击的 markdown 链接，链接文字就是那一章的**真实标题**。绝不用不透明的编号。

| 错误 | 正确 |
|---|---|
| `第 1 部分第 3 章讲过会话保证。` | `[一致性模型：读者到底看到什么](../part-1-core-fundamentals/03-consistency-models.md) 讲过会话保证。` |
| `见第 3.6 章。` | `见 [幂等与精确一次](./06-idempotency-exactly-once.md)。` |
| `[第 1.5 章](../part-1-core-fundamentals/05-back-of-envelope-estimation.md)` | `[量级估算](../part-1-core-fundamentals/05-back-of-envelope-estimation.md)` |
| `（见 8.4）` | `（见 [通知系统](./04-notification-system.md)）` |
| `你在 [数据结构](...) 里学过 B 树。` | `[面向系统的数据结构](...) 讲了 B 树。` |
| `你在 [Quorum](...) 里学过法定人数。` | `[Quorum 与复制](...) 介绍了 quorum 的算术。` |

规则：

1. **链接文字 = 章节标题。** 不写 `Chapter 1.5`，不写 `Part 2 Ch. 4`，不写 `8.4`。标题能告诉读者要去哪，不必让他去背模块编号网格。
2. **有提及就给链接。** 如果一章值得引用，就值得一个链接。“如我们在第 2 部分所见”这种没有链接的裸引用同样糟糕；要么加上链接，要么删掉。
3. **模块编号前缀可选。** 如果某章的位置值得强调（比如在 `## Further Reading` 里），写 `[1.4 CAP 与 PACELC](...)` 也可以。正文叙述里优先只用标题。
4. **同 Part 内的链接用 `./filename.md`；跨 Part 用 `../part-N-name/filename.md`。**
5. **回顾类动词的主语是章节，不是读者。** 写 `[另一章] 讲了 X` / `介绍了 X` / `展示了 X`。不要写 `你在 [另一章] 里学过 X` / `你在 [另一章] 里看到过 X`。前面那些内容不是读者写的，是这本手册写的。第二人称“你”只留给读者当下执行的动作（`你将能够……`、`选取 W 和 R 使……`）。
6. **外部书籍的引用**（如“Designing Data-Intensive Applications, Ch. 6”或“Pro Git, Chapter 10.2”）属于正当的外部引文，不受这条规则约束，因为它们指向的是书，不是手册里的章节。

CI 检查：`scripts/check-cross-references.mjs`（如果存在）会 grep `Part [0-9]+ Chapter`、`\bChapter [0-9]+\.[0-9]+`、`\[Chapter [0-9]` 和 `You (covered|learned|saw|met) .* \[.*\]\(.*\.md\)`，只要在 `content/` 内命中就让构建失败。

### 外部链接

一律用完整 URL（不要短链）。优先一手来源（工程博客、论文、RFC），而不是二手材料（Medium 上的总结）。

```markdown
见 [How Discord stores trillions of messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)。
```

### 引用格式

正文里一律行内引用：`[Author, Year]` 或 `[Company Engineering, Year]`。完整的条目放在“延伸阅读”一节。

### 校验 URL

所有外部 URL 都由 CI 里的 `lychee` 检查。带死链的 PR 会 CI 失败，不会被合并。

## 数字与单位

- 存储用 SI 单位（KB、MB、GB、TB、PB），按十进制理解，除非专门在讨论内存页（那时用 KiB、MiB）。
- 单位一律写出：写 `100 GB`，不写 `100G`。
- 用千分位：写 `1,000,000 QPS`，不写 `1000000`。
- 有效数字要诚实。`~100K ops/sec` 比 `99,847 ops/sec` 更好，除非你确实测出了后者。

引用具体数字时，注明来源：

> Redis 单节点大约能扛 100K ops/sec（[Redis benchmarks, 2023](https://redis.io/docs/management/optimization/benchmarks/)）。

## 标点与排版

### 引号

用直引号（`"` 和 `'`），不要用弯引号（`"` `"` `'` `'`）。多数编辑器会自动替换成弯引号，把这个功能关掉。

### 省略号

用三个 ASCII 句点：`...`，不要用 `…`（U+2026）。

## 无障碍

- 每张图都有描述性的 `alt` 文本。
- 每张图（Mermaid 或 Excalidraw）正下方都有一句斜体图注，说明要点。见[图表](#图表)一节。
- 表格有表头行。
- 链接文字不要写“点这里”；让链接文字本身说明去哪。
- 用 Lighthouse 的无障碍审计跑一遍站点，目标 100 分。

## 反模式

内容里要避免的常见错误：

| 反模式 | 为什么不好 |
|--------------|-------------|
| “Obviously, ...” 或 “Simply, ...” | 没有什么对所有读者都显而易见。 |
| “In 2023, ...” 却没有引用 | 无法核实的说法。 |
| 没有配图的整块文字 | 读者会略读。图能锚住心智模型。 |
| ASCII 方框图（`+---+ \| \|` 那种） | 会和 Mermaid 脱节，在比例字体下错位，而且与 Mermaid 重复。改用 Mermaid 或 Excalidraw。 |
| “You can use Kafka, RabbitMQ, SQS, Pulsar...” 却不推荐其中一个 | 选一个。读者来这里是为了拿决定。 |
| 句子中间用 emoji 表示强调 | 用加粗，不要用 emoji。 |
| 很长的 Markdown 表（>10 行） | 拆分或总结。 |
| 照抄数字却不给引用 | 会毁掉读者的信任。 |
| `## Trade-offs` 的任何单元格里出现 “Never” / “Avoid” / “Almost never” | 破坏了行与行等价的约定。见[权衡表](#权衡表)规则 2。 |
| 表格下方的元评论（“第 N 行是略读者的陷阱”“本表数据截至……”“本节原标题是……”） | 说的是文档，不是主题。改那一行，或者把它挪进 Warning callout。见[权衡表](#权衡表)规则 3。 |
| 同一个事实在 TL;DR、表格、正文、陷阱、要点回顾、记忆卡片里反复重述 | 每个事实出现一两次就够了。重述五遍是过拟合，不是严谨。 |

如果你发现自己正在写其中一种，停下来改掉。如果你在评审里看到一种，留个评论。

---

**拿不准的时候**，找你正在写的内容最接近的现有章节，照着它的风格写。全书一致比个人的文风花活更有价值。
