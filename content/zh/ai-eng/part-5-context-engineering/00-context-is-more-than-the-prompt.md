---
title: "上下文不只是提示词"
description: "每一轮进入上下文窗口的六样东西，为什么提示词工程只覆盖其中一样，以及上下文工程这门手艺究竟负责什么。"
slug: 00-context-is-more-than-the-prompt
part: 5
chapter: 0
difficulty: intermediate
prerequisites:
  - part-0-start-here/03-the-escalation-ladder
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-3-prompt-engineering/00-anatomy-of-a-production-prompt
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 上下文不只是提示词

这个客服 agent 的系统提示词已经一个月没改过了。评测套件全绿。可就在这一周，它引用了一条上季度就已作废的退款政策；因为定义了十一个工具、而只有三个跟这次提问相关，它调错了工具；还对一位用了八个月的老客户说“今天有什么可以帮您？”，好像她从没用过这个产品。团队的第一反应是重写提示词。但这些 bug 没有一个是住在提示词里的。

它们住在上下文（context）里：模型在某一轮实际看到的那一摞拼装好的词元（token）。提示词只是这摞东西里的一层。其余全部——检索回来的政策文档、工具 schema、对话历史、用户存储的偏好、时间戳——都挤在同一个有限的窗口里，争同一份注意力预算。2025 年 6 月 Andrej Karpathy 和 Tobi Lutke 把“上下文工程（context engineering）”这个词推进了公共讨论，他们真正想说的是：大多数 LLM 产品称作“提示词工程”的那份工作，大半与提示词无关。[^1][^2]

## 锚点等式

每一轮都有六个组成部分进入模型的上下文窗口。第 5 部分就是围绕它们搭起来的，接下来的每一章你都会再见到这个等式：

> **上下文（Context）= 提示词（Prompt）+ 记忆（Memory）+ 检索知识（Retrieved Knowledge）+ 工具（Tools）+ 元数据（Metadata）+ 约束（Constraints）**

每个组成部分都是不同的词元来源，由你代码里不同的部位掌管，各自有“什么该进、什么不该进”的规则：

- **提示词（Prompt）** 是系统指令加上用户当前这条消息。经典提示词工程优化的就是它。
- **记忆（Memory）** 是对话状态，以及从某个存储里取出的、关于用户的持久事实。它逐轮变化。
- **检索知识（Retrieved Knowledge）** 是 RAG 流水线为这次具体提问取回的片段。有的提问它为空，有的提问它很密。
- **工具（Tools）** 是模型可以调用的 JSON schema。每个 schema 都花词元，模型用不用都一样花。
- **元数据（Metadata）** 是模型推不出来的环境状态：当前时间戳、用户角色、locale、会话 ID、功能开关。
- **约束（Constraints）** 是显式的护栏：安全规则、拒答策略、格式要求，无论用户问什么都得成立。

![一个有限的矩形上下文窗口，左侧有六条带标签的词元流汇入，右侧只有一支箭头流向模型。](../../../ai-eng/part-5-context-engineering/images/00-context-equation.png)
*六条成分流在任何一个词元到达模型之前，就已经在同一个有限窗口里争夺空间。*

这个等式是记账恒等式，不是配方。它告诉你什么进了窗口，但不告诉你每样占多少、按什么顺序排、放不下时怎么办。那些选择才是这门手艺。

## 为什么“提示词工程”说小了这份工作

“提示词工程”这个词成形于 2022 到 2023 年，围绕的工作流大致是一个开发者、一个静态字符串、一份模型输出。你写一段聪明的提示词，给结果打分，再改措辞。这仍然是一门真本事，[生产提示词的解剖](../part-3-prompt-engineering/00-anatomy-of-a-production-prompt.md) 就是专门讲它的那一章。但它是一门窄本事。

在生产级 agent 里，某一节的措辞并不是承重的那个决策。真正要紧的决策发生在拼装时刻，每一轮都发生，在代码里：

```python
def build_context(user_msg: str, session: Session) -> list[dict]:
    # 记忆：保留、摘要还是丢弃哪些历史轮次？
    history = session.history.last_n_or_summarized(budget_tokens=2_000)

    # 检索知识：这次具体提问该取哪些片段？
    chunks = vector_store.search(user_msg, k=5, filter=session.tenant_id)

    # 工具：23 个可用工具里，哪些适用于这个用户角色？
    tools = registry.tools_for(role=session.user.role, query=user_msg)

    # 元数据：哪些东西模型自己推不出来？
    meta = {
        "now": datetime.now(UTC).isoformat(),
        "user_locale": session.user.locale,
        "user_tier": session.user.tier,
    }

    return assemble(SYSTEM_PROMPT, history, chunks, tools, meta, user_msg)
```

`build_context` 里的每一行都是提示词本身做不了的决策。五个片段挑错了，提示词写得再好也没用。忘了注入时间戳，模型就没法推“本周到期的续约”。23 个工具全传进去，模型就会挑错，因为真正的信号被淹了。Harrison Chase 在 LangChain 那篇博文里的说法最干净：提示词工程是“把你的提示词设计成能在一套输入数据上工作良好”；上下文工程则是拿“一组动态数据，把它恰当地格式化”，好让提示词有像样的东西可处理。[^3]

这就是为什么每一家认真的厂商都收敛到了这个词上。Anthropic 2025 年 9 月的工程博文把上下文工程定义为“在 LLM 推理过程中筛选和维护最优词元（信息）集合的一整套策略，包括提示词之外所有可能落进那里的其他信息”，并明确写道：“如今大多数 agent 的失败是上下文失败，不是模型失败。”[^4] 2025 年 7 月那篇学术综述看了 1400 多篇论文，把同一条区分形式化了：提示词工程优化一个静态字符串，上下文工程组装一个动态的结构化产物。[^5] 提示词工程没有消亡，它是子集。

## 上下文更多不等于能力更强

上下文窗口涨过一百万词元之后，第一反应是把所有东西都塞进去。这个直觉是错的，错的原因写在 transformer 的工作方式里。

自注意力是平方级的：每个词元都要关注其他所有词元，于是两两关系随序列长度按 O(n²) 增长。你把输入拉长，模型的注意力就摊薄到更多词元对上。由此产生两个有充分记录的现象。

第一，**中间失落（lost in the middle）**。Liu 等人 2024 年发表在 TACL 的论文显示，相关信息放在输入开头或结尾时性能最高，放在中间则急剧下降，即便是专门用长上下文训练过的模型也一样。[^6] 这个形状对应人类的首因与近因偏误。实际推论是：一个事实放在提示词里的哪个位置，和它在不在里面一样重要。

第二，**上下文腐坏（context rot）**。Chroma 2025 年 7 月的研究用 18 个前沿 LLM（GPT-4.1、Claude 4、Gemini 2.5、Qwen3）做评测，任务难度固定，只变输入长度。每个模型都随输入变长而退化，连简单任务也不例外。一个干扰项就让准确率低于干净基线，四个干扰项则把损失叠加起来。[^7] Databricks 2024 年做了一项平行研究，形状类似：Llama 3.1 405B 过了 32k 词元就开始掉准确率，GPT-4-0125-preview 过了 64k 开始掉，两者都远低于名义窗口上限。[^8]

Karpathy 的措辞很精确：“用恰好适合下一步的信息填满上下文窗口。”[^1] 不是全部信息，是合适的信息。Anthropic 同脉的原则是：“找到能让某种期望结果概率最大的、尽可能小的高信号词元集合。”[^4] 把上下文窗口当成你在花掉的有限资源，而不是你在灌满的桶。

## 上下文出错的四种方式

后面每一章第 5 部分的内容，底下都是在防四种失效模式之一。Drew Breunig 在 2025 年 6 月给它们起了名字，这些名字留了下来，因为每一个都描述了一个独立、可复现的 bug：[^9]

- **上下文中毒（context poisoning）。** 前面某一轮产生的幻觉被写回上下文（作为工具结果、摘要、记忆笔记），然后在后续轮次被当成事实。Google DeepMind 的 Gemini 2.5 技术报告记录了他们玩 Pokémon 的 agent：“上下文里很多部分（目标、摘要）被关于游戏状态的错误信息‘污染’了，而这种污染往往要很久才能消除。”[^9]
- **上下文分心（context distraction）。** 对话一长，模型过度关注累积的历史，不再动用参数化知识。同一份 Gemini 报告：过了 100k 词元后，agent“表现出偏向从庞大历史里重复既有动作，而不是综合出新的计划”。[^9]
- **上下文混淆（context confusion）。** 无关内容（错的工具 schema、错的检索片段）进了窗口，模型就用了它，因为它在那儿。伯克利函数调用排行榜显示，所有受测模型都随工具数量增加而准确率下降；有个实验里，量化版 Llama 3.1 8B 在 46 个工具全给时失败，只给相关的 19 个就成功了。[^9]
- **上下文冲突（context clash）。** 信息分批到达，后一轮跟前面矛盾；模型已经认下了一个片面答案，就不肯改。微软和 Salesforce 的一项研究把基准提示词切分到多个轮次里，各模型平均准确率掉了 39%。OpenAI 的 o3 从 98.1% 掉到 64.1%。[^9]

这些不是模型 bug，是拼装 bug。每一种都有第 5 部分的一章来治：混淆靠拼装与路由，分心靠压缩与预算，中毒靠记忆设计，冲突靠上下文评估。

## 默认规则

只要你的应用超过一轮，或者要拉任何外部数据，就该立刻用上上下文工程这套纪律。单轮、不检索的场景，纯提示词工程完全够用。一旦加上对话历史、检索或工具，你每一轮都在拼装上下文，而拼装正是最先坏掉的那一环。

三个习惯就能拿到大部分收益。把稳定、高优先级的内容（系统指令、硬约束）放在窗口的开头和结尾，那里注意力最强；易变的内容放中间。工具按请求裁剪，而不是按设计裁剪：每个发给模型、其实用不上的工具 schema，都是一笔词元税，也是一条混淆的通道。还要记录你花了多少：每次拼装出来的上下文都该能写成六个带标签的字节数，因为看不清自己的预算，就管不了它。

这是[上下文拼装与路由](./01-context-assembly-and-routing.md)的起点，那一章把这些习惯变成代码。放到架构尺度上，HLD 第 9 部分的 [AI 系统设计](../../hld/part-9-ai-ml-system-design/) 讲的是白板视角：向量存储、记忆存储，以及针对“什么进入了上下文”的可观测性。

## 参考文献

[^1]: Andrej Karpathy, X post, June 25, 2025, https://x.com/karpathy/status/1937902205765607626
[^2]: Tobi Lutke, X post, June 19, 2025, https://x.com/tobi/status/1935533422589399127 (2.05M views, 881 reposts as of June 2025)
[^3]: Harrison Chase, "The rise of 'context engineering'", LangChain Blog, June 23, 2025, https://www.langchain.com/blog/the-rise-of-context-engineering
[^4]: Anthropic Applied AI team, "Effective context engineering for AI agents", Anthropic Engineering, September 29, 2025, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
[^5]: Lingrui Mei, Jiayu Yao, Yuyao Ge et al., "A Survey of Context Engineering for Large Language Models", arXiv:2507.13334, July 17, 2025, https://arxiv.org/abs/2507.13334
[^6]: Nelson F. Liu, Kevin Lin, John Hewitt et al., "Lost in the Middle: How Language Models Use Long Contexts", TACL 2024, https://arxiv.org/abs/2307.03172
[^7]: Kelly Hong, Anton Troynikov, Jeff Huber, "Context Rot: How Increasing Input Tokens Impacts LLM Performance", Chroma Technical Report, July 14, 2025, https://research.trychroma.com/context-rot
[^8]: Databricks Research, "Long Context RAG Performance of LLMs", Databricks Blog, August 2024, https://www.databricks.com/blog/long-context-rag-performance-llms
[^9]: Drew Breunig, "How Long Contexts Fail", dbreunig.com, June 22, 2025, https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html (citing Gemini 2.5 technical report, Berkeley Function-Calling Leaderboard, and the Microsoft and Salesforce sharding study, arXiv:2505.06120)
