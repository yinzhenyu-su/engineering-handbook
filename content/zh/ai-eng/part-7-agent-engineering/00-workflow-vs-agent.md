---
title: "Workflow 与 agent"
description: "控制流能提前用 Python 写出来，那它就是 workflow；如果由模型在运行时决定，那它就是 agent。生产环境里的大部分收益其实来自 workflow。"
slug: 00-workflow-vs-agent
part: 7
chapter: 0
difficulty: intermediate
prerequisites:
  - part-0-start-here/03-the-escalation-ladder
  - part-2-working-with-model-apis/03-function-and-tool-calling
  - part-3-prompt-engineering/03-templates-and-chaining
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# Workflow 与 agent

2026 年 1 月，Moderna 研究数据科学负责人 Eric Ma 把团队做过的每一个 LLM 应用都审了一遍。他拿 Anthropic 那篇 *Building Effective Agents* 里的同一个标准逐个衡量：下一步发生什么，是模型说了算，还是 Python 说了算？他的结论是：“我们没有 agent。我不会把我们任何一个应用归为 agent。”[^1] Moderna 是一家受监管的药企，LLM 功能已经在生产环境上线。但没有一个是大多数人嘴里说的那个“agent”。

这不是 Moderna 一家的怪事。MAP 研究（Pan 等人，2025 年 12 月，已被 ICML 2026 接收为 Oral）审查了 26 个领域里 86 个已部署的 agent 系统。68% 的系统最多执行 10 步，然后就得人接手。74% 依赖人来评估。它们大多不是开放式循环，而是两头都坐着人的编排流水线。[^2] agent 的炒作曲线和生产现实曲线，本来就不是一个形状。

本章讲的就是把两者分开的那条线，以及由它推出来的一条规则。

## Python 判定法

Anthropic 的定义是这个领域最终收敛到的那个：**workflow** 用预定义的代码路径来编排 LLM 调用和工具。**agent** 让模型动态决定自己的流程和工具使用方式，从环境的回应里判断下一步做什么。[^3] 两者都算“agentic 系统”，但只有后者是 agent。

这个定义没错，只是太抽象。五秒钟就能用上的版本是这样：

> 如果你在看到输入之前就能用 Python 写出控制流，那它是 workflow。如果接下来跑什么由 LLM 的输出决定，那它是 agent。

同一个任务，换两个角度看。下面这个 workflow 给客服工单分类并派发：

```python
def handle_ticket(ticket: str) -> str:
    intent = llm_classify(ticket, options=["billing", "technical", "other"])
    if intent == "billing":
        return billing_handler(ticket)
    if intent == "technical":
        return technical_handler(ticket)
    return general_handler(ticket)
```

LLM 负责分类，Python 负责分支。路径集合可以枚举，可能执行的每一行代码你都能列出来。你可以给它写单元测试。你可以给它定一条延迟 SLA。

同一个任务，写成 agent：

```python
def handle_ticket(ticket: str, tools: dict, max_steps: int = 10) -> str:
    history = [{"role": "user", "content": ticket}]
    for _ in range(max_steps):
        response = llm_with_tools(history, tools=list(tools.keys()))
        if response.tool_call is None:
            return response.text          # 模型自己觉得干完了
        result = tools[response.tool_call.name](**response.tool_call.args)
        history.append({"role": "tool", "content": result})
    return "max_steps reached"
```

`for` 循环还在 Python 里，但它是安全上限，不是调度表。停止条件（`tool_call is None`）是模型的输出。迭代几次、调哪个工具、按什么顺序调，全在运行时由模型决定。同一个函数的两次请求，可能走出完全不同的路径。

结构上的差别就这些。它不在于你调了多少次 LLM，不在于你用不用工具，也不在于系统“感觉起来聪不聪明”。连续五次 LLM 调用的提示链是 workflow。模型在两个工具里挑一个的单次调用循环是 agent。**决定点在于 `if` 归谁写。**

![并排对比示意图：左边是一段 Python 代码，里面有一次 LLM 调用和干净利落的 if/else 分支，通向三个 handler 方框；右边是 LLM 位于一个环形循环的中心，箭头向外散到多个工具方框再绕回来，周围有一圈虚线画出的最大步数环。左侧上方一行粗体说明写着“Python owns the decision”，右侧上方写着“Model owns the decision”。](../../../ai-eng/part-7-agent-engineering/images/00-control-flow-ownership.png)
*同一个任务，两种架构。差别不在 LLM 调用次数，而在于谁决定接下来跑什么。*

## 这条规则

> **默认上 workflow。只有当步骤没法预先枚举、系统能从任务中途的失败里恢复过来、而且跑飞循环的代价和延迟都还扛得住时，才去够 agent。**

这是 Anthropic 的说法，几乎一字不差：“找到最简单的方案，只在必要时才加复杂度。”[^3] 它属于[升级阶梯](../part-0-start-here/03-the-escalation-ladder.md)（the escalation ladder）那条更大的原则：爬上能解决问题的最便宜那一级，然后停在那儿。workflow 就是那把梯子上 agent 下面的一级。

这条规则带着立场，是有原因的。它背后有一个数。

## 误差累积的算术

如果单个 LLM 步骤有 95% 的正确率，且步骤之间相互独立，那么 10 步流水线整体正确的概率是 `0.95^10 = 60%`。20 步掉到 36%。100 步的全程正确率是 0.6%。Chip Huyen 在 *AI Engineering* 里把这个算法直接写了出来：“如果模型每步准确率是 95%，走 10 步，准确率会掉到 60%；走 100 步，准确率只剩 0.6%。”[^4]

这是下限，不是上限。真实 agent 的错误并不独立，它们会在语义上叠加。轨迹前面一次错误的工具调用污染了上下文，模型就一直在一个被下毒的前提上推理。那条算术曲线是*最好的情况*。

这不是纸上推导。Sierra AI 的 tau-bench（2024 年 6 月）在真实的客服 agent 任务上测了 GPT-4o。单次运行通过率（`pass^1`）不到 50%。把*同一个*任务换成八个不同客户跑八次（`pass^8`）：零售场景大约 25%。Sierra 团队的总结很直白：“agent 只有 25% 的概率能把同一个问题在 8 个不同客户身上都解决掉。”[^5] 最前沿的模型，结构化的任务，四个客户里也只有一个能拿到稳定的答案。

基准分数和生产表现之间的差距，比基准本身暗示的更大。SWE-bench Verified 的分数从 2023 年的 4.4% 跳到 2024 年的 71.7%。[^6] 然后 METR（2026 年 3 月）去查了那些通过的 PR，维护者到底会不会真的合并。大约一半不会，即便把合并决策本身的噪声扣掉之后也还是这样。[^7] 通过测试不等于做对了。

workflow 绕开了其中大部分问题，因为步数有界、路径可枚举。你可以把每一步单独加固，再把整条链作为一个整体来测。agent 在构造上就给不了这一点。

## 五种 workflow 撑起了大部分生产收益

Anthropic 那篇文章点名了[五种可组合的 workflow 模式](./01-the-five-workflow-patterns.md)，几乎每个值得研究的生产系统里都能见到。[^3] 下一章会把它们讲透，这里先给地图：

- **提示链（prompt chaining）** 跑一串固定的 LLM 调用，每一步吃前一步的输出。当任务能干净地切成阶段时用它：列大纲、起草、编辑；分类、抽取、格式化。
- **路由（routing）** 给输入分类，再派给专门的提示或模型。当输入类别稳定、而且你能把简单查询丢给便宜模型、把难查询丢给更强模型时用它。
- **并行化（parallelization）** 把独立子任务扇出（分区 sectioning），或者把同一个任务跑 N 次再投票（投票 voting）。用在护栏、多视角评审、或者提升置信度上。
- **编排者-工作者（orchestrator-workers）** 让一个 LLM 把任务拆成子任务，在运行时派给 worker LLM。外层循环还是代码，拆分不是。Anthropic 自己的 SWE-bench 编码 agent 就住在这里。
- **评估者-优化者（evaluator-optimizer）** 把生成 LLM 和评估 LLM 配成一对循环，迭代到评估者点头为止。用在翻译、复杂检索，以及任何反馈真的可执行的地方。

真实产品通常是其中两三种缝在一起。Ma 那次审计里，Moderna 价值最高的 LLM 应用就是提示链和编排者-工作者，跑在科学文献和归档表单上。[^1] 没有 agent。真金白银。

[agent 循环本身](./02-the-agent-loop.md)（ReAct、Reflexion、规划、反思）单独一章，因为这五种模式不够用的时候，你要去拿的就是那个架构。

## 规则什么时候说“该用 agent”

别把“workflow 优先”读成“agent 是错的”。有些问题确实需要在运行时选路径，硬编码它们只会做出脆弱的花架子流水线，撞上开发者没预料到的第一个输入就崩。

真的撞上一个 agent 任务的信号是：

- **步数取决于输入**，而且你没法枚举。一个修 bug 的编码 agent 可能只碰一个文件，也可能碰十五个；模型读完代码库之前你不知道。Anthropic 自己的 SWE-bench agent 就是这种情况。[^3]
- **任务中途的失败需要重新规划，而不是重试。** 第三步失败时，下一步该做什么取决于它*为什么*失败。如果这段恢复逻辑你写不出来，那就得让模型来写。
- **环境是沙箱的、可回滚的。** 让 agent 直接写生产数据库又没有回滚路径，是团队在凌晨三点发现误差累积的方式。先只读或沙箱；写操作只放在明确的确认之后。
- **延迟和成本预算容得下一个循环。** agent 会把 LLM 账单乘上步数，还会给每个请求加上好几秒。如果你在服务一个 2 秒 SLA 的交互式聊天，agent 不可行；如果你是夜里批量处理工单，那也许可行。

这四条缺任何一条，你要的就是带人工升级的 workflow——把不合套路的输入交给人，而不是 agent。编排者-工作者模式常常是合适的折中：子任务由 LLM 在运行时挑，但外层循环的停止条件留在 Python 里。

## 诚实的分歧

“workflow 优先”是当下这一刻的建议，不是永久定律。基座模型变强，阈值就会移动。OpenAI 的 *Practical Guide to Building Agents*（2025 年 4 月）站得软一些：agent“特别适合那些传统确定性方法和规则式方法搞不定的流程”。[^8] LangChain 的 Harrison Chase 强烈反对，说这份指南“方向错了”，并论证 workflow 系统“在底层模型升级时能保住价值”，而端到端 agent 往往需要重写。[^9]

OpenAI 的 Hyung Won Chung 给出了更长周期的反驳：加 workflow 结构能拿到短期收益，但模型规模上去后往往会失地。[^9] 这是把苦涩的教训（Bitter Lesson）搬到系统架构上：你手工搭进去的结构，最终会被更强的模型吃掉。时间线拉得足够长，他大概是对的。但在今天，在我们手上的生产数据上，他不对。

MAP 研究、Moderna 的审计、tau-bench 的数字，都指向同一个方向：2026 年年中，在你能真正从生产代码里调用的那些模型上，workflow 能上线，agent 很挣扎。等这变了，规则也会跟着变。在那之前，默认答案没有歧义。

如果你的团队正在伸手去够 agent，只是因为它听起来像那个更严肃的答案，那正是该问一句“我们能不能先用 Python 把控制流写出来”的时刻。大多数时候，能。

放到架构尺度上，HLD 第 9 部分的 [AI system design](../../hld/part-9-ai-ml-system-design/) 讲的是这个区分底下的基础设施：沙箱执行环境、工具注册表，以及 agent 轨迹的可观测性。

## 参考文献

[^1]: Eric J. Ma and Hugo Bowne-Anderson, "Episode 66: The Agent Paradox - Why Moderna's Most Productive AI Systems Aren't Agents," Vanishing Gradients podcast, January 8, 2026. https://hugobowne.substack.com/p/episode-66-the-agent-paradox-why
[^2]: Melissa Z. Pan, Negar Arabzadeh, et al., "Measuring Agents in Production," arXiv:2512.04123v4, December 2025 (revised June 2026), accepted as Oral at ICML 2026. https://arxiv.org/abs/2512.04123
[^3]: Erik Schluntz and Barry Zhang (Anthropic), "Building Effective Agents," Anthropic Engineering Blog, December 19, 2024. https://www.anthropic.com/engineering/building-effective-agents
[^4]: Chip Huyen, "Agents," huyenchip.com, January 7, 2025; adapted from *AI Engineering: Building Applications with Foundation Models*, O'Reilly, 2025. https://huyenchip.com/2025/01/07/agents.html
[^5]: Karthik Narasimhan et al. (Sierra AI), "tau-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains," arXiv:2406.12045, June 2024. https://arxiv.org/abs/2406.12045
[^6]: Stanford HAI, "The 2025 AI Index Report: Technical Performance," Stanford University, 2025. https://hai.stanford.edu/ai-index/2025-ai-index-report/technical-performance
[^7]: METR, "Many SWE-bench-Passing PRs Would Not Be Merged into Main," METR Notes, March 10, 2026. https://metr.org/notes/2026-03-10-many-swe-bench-passing-prs-would-not-be-merged-into-main/
[^8]: OpenAI, "A Practical Guide to Building Agents," OpenAI Business Guides, April 2025. https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents
[^9]: swyx (Shawn Wang), "In the Matter of OpenAI vs LangGraph," Latent Space, April 20, 2025. https://www.latent.space/p/oai-v-langgraph
