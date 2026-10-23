---
title: "上下文组装与路由"
description: "为什么 100 万 token 的窗口不等于免费的上下文，以及先路由、再组装这条流水线如何决定模型真正看到什么。"
slug: 01-context-assembly-and-routing
part: 5
chapter: 1
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-2-working-with-model-apis/07-prompt-caching
  - 00-context-is-more-than-the-prompt
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 上下文组装与路由

2025 年的两篇论文给出了几个数字，本来应该杀掉一个流行说法。NoLiMa（ICML 2025）在长上下文检索任务上测了 13 个前沿模型，题目和“针”几乎不含字面重叠：仅仅 32,000 个 token，13 个里就有 11 个掉到短上下文基线准确率的 50% 以下。表现较好的 GPT-4o 也随着上下文填满，从 99.3% 跌到 69.7%。[^1] 另一项关于有效上下文窗口的研究发现，有些任务配置在上下文里只有 100 个 token 时就已经崩了。[^1a]

死掉的那个说法是“我们现在有 100 万 token 的窗口了，上下文工程可以收工了”。并没有。窗口变大了，模型用满整个窗口的能力没变大。花 $3 跑一次 GPT-4.1、塞进一百万你没筛过的 token，买到的只是一个说得信心十足的错误答案，而且很慢。Anthropic 现在把这条告诫写进了自己的文档，管这个效应叫“上下文腐化（context rot）”：token 数一涨，准确率和召回率就往下走。[^2]

所以工程问题不是“能装下什么”，而是“里面应该放什么”。上下文组装（context assembly）就是在每次调用之前，决定到底哪些 token 占据窗口这件事。路由（routing）是上游的闸门：先给请求分类，再为这类请求组装上下文。两步，顺序固定，每次调用都走一遍。

## 四个层级，一份预算

把窗口当成一份四层的预算，每层有自己的淘汰规则。在生产里摸爬过的从业者收敛出来的模型是这样的：[^3]

- **第 1 层，静态锚点。** 系统提示词和工具 schema。这些永不淘汰。它们待在提示词最前面，很少变动，是你手上最容易被缓存的内容。把它们控制在窗口的 15% 左右。20 万 token 的窗口配 4 万 token 的系统提示词，用户还没打字就已经烧掉 20% 了。
- **第 2 层，检索上下文。** RAG 片段、注入的记忆、刚取回来的文档。按需取用，先淘汰最旧的。别预加载整份文档；留着句柄，按需取片段。一个工具往只剩 2.2 万 token 余量的 agent 里返回 2 万 token 的原始 JSON，任务就被它卡死了。
- **第 3 层，对话历史。** 滚动压缩，先淘汰最旧的。你不管它，历史就会一轮一轮永远涨下去。agent 会话到第 30 轮，它会占掉你 80% 的预算。
- **第 4 层，暂存空间。** 已经消费过的工具输出、中间推理、思维链轨迹。狠狠地淘汰。Claude 的 API 已经在轮次之间自动剥掉扩展思考块，这个默认就是对的。[^2]

一个合理的起始分配：15% / 40% / 25% / 20%。按你的流量调，但上线前就得把比例定死。“合理分配”不是一条规则。

![一张纵向四段堆叠图，从上到下展示上下文预算的各个层级：最上方窄而浅的一段是第 1 层系统提示词与工具，中上部最宽的一段是第 2 层检索上下文，中间一段是第 3 层对话历史，最下方一段是第 4 层暂存空间；左侧一个向下的箭头标着优先级递减，右侧标着各层的淘汰说明](../../../ai-eng/part-5-context-engineering/images/01-context-budget-stack.png)
*四层预算。顶部不动，底部先被淘汰。*

落到代码里，这就是一个带优先级的装箱问题。每层有自己的上限；你从优先级最高的往下填，淘汰则按各层自己的规则来。经得起大多数重构的形状长这样：

```python
from dataclasses import dataclass

@dataclass
class ContextBudget:
    system_prompt_max: int
    retrieved_max: int
    history_max: int

def assemble(
    system_prompt: str,
    retrieved_chunks: list[str],
    history: list[dict],
    budget: ContextBudget,
    tokenize,  # 可调用对象：str -> int
) -> dict:
    if tokenize(system_prompt) > budget.system_prompt_max:
        raise ValueError("System prompt exceeds its hard budget")

    # 第 2 层：检索片段，从得分最高的开始取
    kept_chunks, used = [], 0
    for chunk in retrieved_chunks:
        t = tokenize(chunk)
        if used + t <= budget.retrieved_max:
            kept_chunks.append(chunk)
            used += t

    # 第 3 层：历史，从后往前遍历，先淘汰最旧的
    kept_history, used = [], 0
    for msg in reversed(history):
        t = tokenize(str(msg))
        if used + t > budget.history_max:
            break  # 最旧的消息被丢掉
        kept_history.insert(0, msg)
        used += t

    return {"system": system_prompt, "retrieved": kept_chunks, "messages": kept_history}
```

这段代码除了逻辑本身，还有两点值得说。第一，每层都有自己的上限，而且是提前定好的。历史层吃不掉检索层，因为它们不共用计数器。第二，淘汰是显式的、可观测的：记下哪些片段被丢了，记下哪些历史消息被切掉了。下周模型答错时，你就是靠这份日志发现“相关的那一轮在三个步骤之前就被淘汰了”。

## 放在哪里，和放什么一样重要

第二个发现值得单独一节，因为大多数工程师把检索到的片段一股脑塞在提示词中间，还意识不到自己干了什么。Liu 等人（TACL 2024）用 20 篇检索文档做多文档 QA，做了一次受控实验。相关文档放在第 1 位或第 20 位时准确率最高；同一篇文档放到第 5 到第 15 位之间，准确率掉超过 30 个百分点。这个效应在他们测过的每个模型上都成立，包括那些以“长上下文”为卖点的。[^4] Chroma 2025 年的研究在更新的前沿模型上确认了同样的形状。[^5]

Transformer 的注意力呈 U 形：上下文开头和结尾拿到的权重比中间大。所以组装顺序不是装饰，而是：

1. **系统提示词与工具放最前。** 稳定、可缓存，而且落在高注意力区。
2. **压缩后的历史放中间。** 按设计，信号最低的内容就放这儿，因为中间是注意力最低的区域。
3. **检索片段紧贴用户轮次之前。** 近因效应说了算；模型最后读到的东西拉力最强。
4. **用户最新的一条消息放最底部。** 永远最后。

如果你加载了 20 个检索片段，最相关的那一个最糟的位置就是第 10 位。先按相关度得分给片段排序，再把得分最高的放到离用户轮次最近的地方：最后进入，最先被读到。这不是什么边角技巧。在 Liu 等人的实验设置里，这就是 75% 准确率和 45% 准确率的差别。

## 先路由，再组装

到这里我只描述了一份预算、一种组装顺序。生产里你有好几套，因为不同类型的请求需要结构上不同的上下文。一个事实查询要的是紧凑的知识库检索配方，历史几乎不要。一句随意的问候几乎不需要检索。一个改代码的请求要完整文件加最近的编辑历史，通用文档一律不要。

所谓**上下文配方（context recipe）**，就是其中一套的规格：查哪个检索后端、暴露哪个工具子集、留多少历史、用哪个系统提示词变体。路由是在组装跑起来之前挑配方的上游闸门。架构是两级，有先后：先分类，再组装。

三种路由机制几乎能覆盖所有情况。

**基于规则的路由。** 一个正则或模式匹配。零延迟，完全确定。查询里出现 URL 就走网页抓取配方，出现 SQL 关键字就走数据库配方。把它当作高频、定义明确意图的第一道过滤。规则命中了，就不用往下走。

**语义路由。** 每条路由用几句示例话语来定义。启动时嵌入一次。请求到来时，嵌入查询，取余弦相似度最高且超过阈值的那条路由。用 API 编码器的话，决策延迟大约 100 ms，token 成本基本为零（一次嵌入调用，不做 LLM 补全）。[^6] aurelio-labs 的 `semantic-router` 库是参考实现。[^7]

**LLM 分类路由。** 把查询连同一串路由描述发给一个小模型，让它来挑。LangChain 的 `RunnableBranch` 和 LlamaIndex 的 `RouterQueryEngine` 都是这个形状。[^8] 决策延迟在 200 到 2000 ms，而且你要为分类器那次调用付钱。当路由之间的区别细微到嵌入相似度分不开时，用它。想想客服产品里的“投诉 vs 功能请求”，差别在语气，不在词汇。

默认规则：从语义路由起步。它比 LLM 版本快一个数量级，成本几乎为零，路由数在 20 条以内、语义又足够可分时它工作得很好。只有当某几条路由在你的流量上被语义路由可测量地判错时，才把那几条升级到 LLM 路由。基于规则的路由可以叠在两者之上，作为确定性意图的快速预过滤。别一次把三种都建起来；攒够了足以支撑它们的失败案例，再一个个加。

一个最小的语义路由器，配方挂在每条路由上：

```python
from dataclasses import dataclass
from typing import Callable, Optional
import math

@dataclass
class Route:
    name: str
    utterances: list[str]
    recipe: dict  # 例如 {"retrieval": "kb_docs", "tools": ["search"], "history_turns": 4}

def cosine(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    norm = math.sqrt(sum(x*x for x in a)) * math.sqrt(sum(x*x for x in b))
    return dot / (norm + 1e-9)

class SemanticRouter:
    def __init__(self, routes: list[Route], embed: Callable, threshold: float = 0.75):
        self.routes = routes
        self.embed = embed
        self.threshold = threshold
        self._index = {r.name: [embed(u) for u in r.utterances] for r in routes}

    def pick(self, query: str) -> Optional[Route]:
        q = self.embed(query)
        best, score = None, 0.0
        for r in self.routes:
            s = max(cosine(q, e) for e in self._index[r.name])
            if s > score:
                best, score = r, s
        return best if score >= self.threshold else None
```

这里有两个细节，人们老是搞错。阈值不是因为它是个魔法数字才取 0.75，这个值要你在标好的数据集上校准出来，每条路由 50 到 100 条查询。太低，不相关的查询会撞进错误的路由；太高，真实查询会掉出去。第二，没匹配上时返回 `None` 是特性，不是 bug。兜底配方（小检索、默认工具）接住长尾。把每条查询都硬塞进*某个*类别，只会保证其中一部分拿到错误的上下文。

## 这套东西在生产里怎么坏

失败模式足够可预测，可以一条条列出来，而且每条都对应一个具体的对策。

**静默截断。** 老一些的模型 API 和粗糙的框架在窗口填满时会丢掉最旧的消息，而且不报错。agent 继续跑，但它三个轮次前改过的文件路径已经从它的记忆里消失了。检测办法是按调用记录 `usage.input_tokens`；突然在模型的硬上限处走平、又没有报错，那就是信号。修复办法是换一个溢出时会大声报错的模型。Claude Sonnet 3.7+ 会返回 `stop_reason: "model_context_window_exceeded"` 而不是静默截断[^2]，而且你应该在碰到硬上限之前就先撞到自己的软预算。

**上下文中毒。** 一份接近填满的上下文会劣化模型的注意力质量，它编出一个听起来很合理的幻觉，这个幻觉又被追加进历史，被之后每一轮当成事实。修法是在步骤之间加校验闸门：没有检查就不接受 agent 生成的中间事实；当质量信号（输出校验失败率、工具错误率）开始抬头时，换一个全新的上下文窗口。[^3]

**路由校准错误。** 阈值太低，不相关的查询被塞进错误的配方；阈值太高，真实查询掉进兜底。从外面看，两种情况都像是“模型很笨”。把每次路由决策连同得分一起记下来，每周抽几百条看，对着真实标签调阈值。这是整条流水线里最便宜的生产收益；几乎没人做。

**工具 schema 膨胀。** 每个请求都暴露 30 个工具，用户还没开口就先吃掉 5-15K token 的 schema。大多数请求只需要其中三个。用你为上下文配方已经搭好的那套路由机制，按请求类型只暴露相关的工具子集。LlamaIndex 的 `ToolRetrieverRouterQueryEngine` 是这个问题现成的实现。[^8]

> [!WARNING]
> 最常见的一个陷阱是历史无界增长。agent 会话到第 30 轮，原始历史能占掉你窗口的 80%，而检索上下文被挤在底部饿死。给历史层设一个硬上限（经验法则里的那个 25%），撞到上限时触发摘要，而不是淘汰。下一章[压缩与上下文预算](../../../ai-eng/part-5-context-engineering/02-compression-and-context-budgets.md)是完整论述。

让这一切变得可控的习惯，是把每次调用组装出来的上下文当成带版本的产物记下来：哪条路由命中了、哪些片段进了、哪些被淘汰、各层 token 计数是多少。客户报来一个错误答案时，你能还原模型当时到底看到了什么。没有这份日志，你就是在猜一个概率分布。

## 参考文献

[^1]: Ali Modarressi et al., "NoLiMa: Long-Context Evaluation Beyond Literal Matching," ICML 2025 (PMLR 267). arXiv:2502.05167. https://proceedings.mlr.press/v267/modarressi25a.html
[^1a]: Norman Paulsen, "Context Is What You Need: The Maximum Effective Context Window for Real World Limits of LLMs," arXiv:2509.21361, September 2025. https://arxiv.org/abs/2509.21361
[^2]: Anthropic, "Context windows," Claude Developer Platform documentation, accessed June 2026. https://docs.anthropic.com/en/docs/build-with-claude/context-windows
[^3]: Tian Pan, "Tokens Are a Finite Resource: A Budget Allocation Framework for Complex Agents," TianPan.co, April 2026. https://tianpan.co/blog/2026-04-17-token-budget-allocation-complex-agents
[^4]: Nelson F. Liu et al., "Lost in the Middle: How Language Models Use Long Contexts," Transactions of the Association for Computational Linguistics (TACL), vol. 12, 2024 (accepted November 2023). arXiv:2307.03172. https://arxiv.org/abs/2307.03172
[^5]: Kelly Hong, Anton Troynikov, Jeff Huber, "Context Rot: How Increasing Input Tokens Impacts LLM Performance," Chroma Technical Report, July 2025. https://research.trychroma.com/context-rot
[^6]: truto.one, "How to Implement Semantic Routing for AI Agents to Select API Endpoints," April 2026. https://preview.truto.one/blog/how-to-implement-semantic-routing-for-ai-agents-to-select-api-endpoints/
[^7]: aurelio-labs, "Quickstart," Semantic Router documentation, accessed June 2026. https://docs.aurelio.ai/semantic-router/get-started/quickstart
[^8]: LlamaIndex, "Routers," developer documentation, accessed June 2026 (https://developers.llamaindex.ai/python/framework/module_guides/querying/router/); LangChain, "How to route between sub-chains," accessed June 2026 (https://python.langchain.com/docs/how_to/routing).
