---
title: "多智能体系统"
description: "多智能体拓扑什么时候值它那 15 倍的 token 成本，什么时候不值，以及那套能覆盖大多数生产工作的单写者模式。"
slug: 09-multi-agent-systems
part: 7
chapter: 9
difficulty: advanced
prerequisites:
  - 00-workflow-vs-agent
  - 02-the-agent-loop
  - 03-tool-design
  - 04-mcp-model-context-protocol
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 多智能体系统

Anthropic 自己 2025 年 6 月的生产数据：一个多智能体系统花的 token 大约是普通聊天调用的 15 倍，带工具的单智能体大约是 4 倍。[^1] 也就是说，从你开出第二个 agent 的那一刻起，同一个查询就比交给一个 agent 跑贵了约四倍，比直接问模型贵了约十五倍。这个倍数就是本章存在的全部理由。多数团队听到“多智能体”，脑子里出现的是并行和涌现出来的智能。诚实的账是这样：以前你为一个上下文窗口付钱，现在要付十五个，而且最好有个理由。

理由确实有，只是不常有。当任务真的需要比一个上下文窗口装得下的更多信息，而子任务之间真的彼此独立时，多智能体就是正确选择，别的办法都不管用。这些条件不成立时，你买到的是复杂度和 bug。

## 15 倍的成本是怎么堆出来的

这个倍数是结构性的，不是靠工程手段就能消掉的浪费。每启动一个子智能体，就新开一个上下文窗口，里面装着编排者（orchestrator）给的任务描述、共享记忆，以及子智能体运行过程中攒下的全部工具结果。3-5 个子智能体各自拉进 40K-80K token 的搜索结果，返回一份浓缩摘要；编排者再读一遍这些摘要，写出报告；然后单独一轮引用核查再读原始文档去核实每条断言——一次用户查询，你付了大约十五个满上下文窗口的钱。

![一张横向条形图，三根堆叠条分别展示聊天调用为 1 倍基线、单智能体为 4 倍、多智能体为 15 倍，每根条切分成段以显示 token 花在哪里](../../../ai-eng/part-7-agent-engineering/images/09-token-multiplier-stack.png)
*15 倍不是浪费；十五个上下文窗口本来就要这么多钱。*

叠加的风险比这个基线更糟。一个行为失控的子智能体递归地开出更多子智能体，或者一个返回超大结果的工具，都能在任何护栏触发之前把 15 倍再乘上 10 倍甚至更多。[^1] 公开的 bug 报告记录的正是这种形状：Claude Code 的一个缺陷让子智能体可以递归开子代、没有深度限制，一次会话烧掉 80 万以上 token，几乎什么也没产出。[^2] 结论是 15 倍只是地板，不是天花板；一个没有单次运行成本上限的多智能体系统，从设计上就是无界的。

另一面同样真实：在 Anthropic 内部的研究评测上，用 Claude Opus 4 当主、Claude Sonnet 4 当 worker 的多智能体配置，比单个 Claude Opus 4 智能体高出 90.2%，复杂查询的墙钟时间最多缩短 90%。[^1] 任务真是广度优先时，这笔账算得过来。问题在于哪些任务是。

## 什么时候多智能体才值这个价

三个条件，缺一不可：

1. **信息真的装不下。** 一个 20 万 token 的上下文窗口装不下“找出标普 500 里每家 IT 公司的每一位董事会成员”（65 家以上公司，每家一条独立的申报链条）。单智能体在任务结束之前就把上下文用完了。这是唯一一条*强制*多智能体的条件，另外两条说的是这笔钱值不值得花。
2. **子任务可以被证明是独立的。** 子任务 B 不能在同一波里把子任务 A 的输出当输入。如果它们有先后依赖，并行就退化成串行执行外加一层编排开销。这个检验很残酷：把每个子任务的输入写下来，不许引用其它子任务的输出。写不出来，并行就是假的。
3. **任务价值撑得起 15 倍的 token 成本。** 一次聊天调用花 $0.03，多智能体每次查询就是 $0.45。一百万次查询就是 $450K 而不是 $30K。大多数产品功能背不动这个数。

任何一条不成立，单智能体就是正确答案。[^3] Anthropic 自己在《Building Effective Agents》（2024 年 12 月）里的建议一模一样：“我们建议找到尽可能最简单的方案，只在需要时才加复杂度。这可能意味着根本不做 agentic 系统。”[^3]

三条同时成立最清楚的地方是广度优先的研究。要查的独立来源很多，它们的并集超出任何一个上下文窗口，而查询的答案值真金白银。最清楚失效的地方是绝大多数软件工程。编码任务里真正能并行的子任务更少，而动作背后带的隐式决策往往互相冲突。[^1]

## 并行写为什么失败：隐式决策问题

Cognition 2025 年 6 月那篇《Don't Build Multi-Agents》用一个例子把这种失效模式讲透了。[^4] 任务：做一个 Flappy Bird 克隆。两个子智能体并行开工。子智能体 1 做出一个超级马里奥风格的背景；子智能体 2 做出的小鸟和原版游戏的画风对不上。两个子智能体在技术上都完成了自己分到的子任务。合起来是坏的，因为哪个 agent 都没有看到足够完整的原始设定，抓不住那条隐式的美术约束。

这个机制是结构性的，不是提示词写得好不好的问题。编排者拆任务时，对“每个子智能体需要知道什么”做的是有损决策。它没法把攒下的全部上下文转发下去，那等于取消了独立上下文窗口的意义。子任务描述里每省掉一块，都成了假设冲突的潜在来源。模型没有在“跨 agent 沟通很重要”的环境里训练过，所以不会去补这个缺口；它们只是各自自信地做出彼此对不上的选择。[^4]

Anthropic 在生产里撞过同一个失败。接到一个含糊的委派——“研究一下半导体短缺”——之后，他们的一个子智能体去查 2021 年的汽车芯片危机，另外两个则在 2025 年的供应链上重复劳动。[^1] 修法是在编排者的提示词里写明任务边界：目标、输出格式、工具清单，以及一句明确的“不要研究 X，那是另一个子智能体的活儿”。把子智能体的任务描述当 API 契约来写，而不是当自然语言请求。

NeurIPS 2025 的 MAST 分类法在七个框架的 1,600 条以上多智能体执行轨迹上量出了同一幅图景：41.77% 的失败是规格问题，36.94% 是协作失败，21.30% 是校验缺口。[^6] 规格加协作占了全部失败的 79%，而两者都归到同一个根上：agent 在按没能熬过任务分解的假设行事。

由此掉出来的一条原则：**单写者纪律（single-writer discipline）**。一个范围内所有的写都归一个 agent 独有。其它 agent 只贡献分析，永远不做改动。本章后面所有内容都是这条规则的下游。

## 单智能体加子智能体：今天真正可行的模式

多数生产团队要上多智能体是出于错的原因（并行、“智能”），而正确的模式比他们预期的窄。Cognition 2026 年 4 月的更新，写在他们那篇“别建多智能体”的文章十个月之后，记录了三种在生产里稳定跑得通的配置。[^5] 三种有一个共同点：写保持单线程，多出来的 agent 贡献的是判断力，不是动作。

**生成者-校验者循环（generator-verifier）。** 一个主智能体写代码；另一个独立的评审智能体从一个干净的上下文窗口读 diff，报 bug。Cognition 在 Devin 的 PR 上跑这套，报告平均每个 PR 抓到 2 个 bug，其中 58% 是严重的（逻辑错误、漏掉的边界情况、安全漏洞）。[^5] 反直觉的设计选择是那个干净上下文：评审者不和写代码的共享历史。长上下文会拉低注意力质量（Cognition 称之为“上下文腐化”），所以一个干净上下文的评审者对 diff 的判断，比写代码的人对自己作品的判断更准。[^5]

**请教更强的模型（smart-friend escalation）。** 一个更便宜的主模型在碰到超出自己信心范围的事情时，把更强的模型当工具调一次。强援只贡献分析；所有改动由主模型做。两个模型都是前沿档时这套能用，主模型明显更弱时就会崩，因为弱模型判断不准什么时候该升级。[^5]

**带范围划分的管理者-工作者。** 一个管理者 agent 把大任务切成互不重叠的范围（不同服务、带清晰 API 边界的不同模块），按范围开子 agent。单写者纪律靠每个子 agent 拥有不同的文件集来成立。这是三种里唯一让子 agent 真去写的模式；约束是范围不重叠，所以写不冲突。

这个模式最小的样子是这样：

```python
import anthropic

client = anthropic.Anthropic()

def query_codebase(question: str, codebase_path: str) -> str:
    """Read-only subagent. Answers questions, never writes."""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        system=(
            "You are a read-only code analysis agent. "
            "Answer questions about the codebase. Never propose changes."
        ),
        messages=[{"role": "user", "content": f"{codebase_path}\n\n{question}"}],
    )
    return response.content[0].text

def primary_agent(task: str, codebase_path: str) -> str:
    """Single writer. Calls the read-only subagent for context."""
    context = query_codebase(
        "What are the main interfaces and their contracts?", codebase_path
    )
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        system="You are a senior engineer. Use the provided context to complete the task.",
        messages=[{"role": "user", "content": f"Task: {task}\nContext: {context}"}],
    )
    return response.content[0].text
```

只读子智能体在结构上就是一次工具调用。它没有副作用，所以引入不了冲突的决策。写权限全部留在主智能体手里。你付的可能是 2 倍聊天 token 而不是 15 倍，同时躲掉了那一整类搞坏并行写者集群的失败。

这就是编码 agent 的默认答案。编排者-工作者模式该在——且只该在——你确认过那三个“多智能体合理”的条件、且任务是广度优先的研究而不是软件工程时动用。

## 编排者-工作者模式，确实需要的时候

Anthropic 的 Research 功能是教科书级的生产案例。一个 LeadResearcher（Claude Opus 4）把查询拆成 3-5 个并行子任务，派给 Sonnet 4 子智能体，每个在自己的上下文窗口里跑搜索，最后综合结果。[^1] 骨架：

```python
import anthropic
import asyncio

client = anthropic.AsyncAnthropic()

async def lead_plan(query: str) -> list[str]:
    response = await client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=(
            "You are a lead researcher. Given a query, return a JSON list of "
            "independent subtasks. Each must be completable without knowledge "
            "of the others. Include explicit boundaries: what each subagent "
            "must NOT research because another subagent owns it."
        ),
        messages=[{"role": "user", "content": query}],
    )
    return parse_subtasks(response.content[0].text)

async def subagent(subtask: str) -> str:
    response = await client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        system="Research subagent. Complete your assigned task. Do not exceed scope.",
        messages=[{"role": "user", "content": subtask}],
    )
    return response.content[0].text

async def run(query: str) -> str:
    subtasks = await lead_plan(query)
    findings = await asyncio.gather(*[subagent(s) for s in subtasks])
    synthesis = await client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        system="Synthesize parallel findings into one cited report.",
        messages=[{"role": "user", "content": str(findings)}],
    )
    return synthesis.content[0].text
```

有三个细节比它们看上去更重要。第一，主干用的模型比 worker 更强：Opus 负责编排，Sonnet 负责执行。成本压在 worker 上，判断力留在顶上。第二，规划提示词明确要求子任务之间划清边界；含糊的委派正是 Anthropic 早期生产里烧掉一堆钱的那个失效模式。第三，综合那一步读的是子智能体的输出，不是原始来源，这就埋下了下一个问题。

从摘要做综合这个问题，Anthropic 叫它传话游戏式失败（game-of-telephone）。等报告起草时，来源 URL 已经过好几轮子智能体返回被压缩、再摘要，主干只能从被压缩过的记忆里重建引用。[^1] 他们的修法是单独一个 CitationAgent，同时读原始来源文档和最终报告，逐条核实断言。这个修法的通用形态是：子智能体把大块输出写到外部存储，只把轻量引用传回编排者；编排者在综合时读一手产物，不读子智能体摘要。

另一个生产陷阱是无界的开销。每次 agent 调用都设 `max_turns` 和 `max_budget_usd`。禁止子智能体递归派生，除非你专门为它设计过；Anthropic 早期有个 agent 在一个简单查询上开了 50 个子智能体，之后才加上规模规则。[^1] Anthropic 的经验法则：简单事实查找用 1 个 agent 加 3-10 次工具调用，只有真正复杂的研究才上 10 个以上子智能体。[^1]

## A2A：知道有这回事就够了

Agent2Agent（A2A）是跨组织 agent 委派的开放协议。Google 2025 年 4 月公布，同年晚些时候捐给 Linux 基金会，到 2026 年 4 月已有 150 多家支持组织，并在 Microsoft Azure AI Foundry、Amazon Bedrock AgentCore Runtime 和 Salesforce Agentforce 3 里有生产部署。[^7] 当前规范是 v1.0，第一个稳定版本，2026 年 3 月发布。[^8]

三件事记住就够了，等你真需要它的时候再说。

**A2A 不是 MCP 的竞争对手，它是上面那层。** [Model Context Protocol (MCP)](04-mcp-model-context-protocol.md) 规范的是 agent 怎么跟自己的工具说话（纵向，agent 对工具）。A2A 规范的是 agent 之间怎么把活儿委派出去（横向，agent 对 agent）。A2A 的文档自己就划了这条线：MCP 是连接 agent 与工具、资源的推荐协议，A2A 是连接 agent 与其它 agent 的协议，一个功能完整的 agent 应该两种都说。[^8] 两个协议现在都归 Linux 基金会，被当作一对配套标准。

**机制是三个原语。** Agent Card（`/.well-known/agent-card.json` 上的 JSON 文档）声明一个 agent 的技能、端点和认证方式。任务信封（HTTPS 上的 JSON-RPC 2.0）携带唯一任务 ID 和技能调用。任务生命周期状态机（`submitted -> working -> input-required -> completed | failed | canceled | rejected`）控制多轮交互。[^8] 调用方永远看不到被调 agent 的提示词、模型、工具和记忆；两边只交换任务和结果。

**跨组织或跨框架边界时才需要它。** 一个 Salesforce 的 agent 去调一个 Microsoft Copilot 的 agent。一个面向外部发布的专用 agent。不同团队各自拥有不同 agent，必须在不紧耦合的前提下互通。在一个代码库里、所有 agent 共用同一个框架时，A2A 是你不需要的开销；直接函数调用或 MCP 工具调用更简单也更好控。这个决定跟协议质量无关，只跟你有没有一个线路格式（wire format）问题有关。

整章可以压成一个默认加一个例外。默认：单 agent、单写者，只读子智能体随便加，因为它们结构上就是工具调用。例外：编排者-工作者加并行子智能体，只在信息真的超出单个上下文窗口、子任务可证明独立、答案值十五个上下文窗口的 token 时才用，而且不管哪种都要给每次派生设预算上限。[Agent 原型](10-agent-archetypes.md)梳理了这些模式在团队真正跑的系统里长什么样；它们底下的编排基础设施——队列、检查点、扇出——在架构尺度上是 HLD 手册[多智能体编排一章](../../hld/part-9-ai-ml-system-design/04-multi-agent-orchestration.md)的内容。

## 参考文献

[^1]: Anthropic, "How we built our multi-agent research system," Anthropic Engineering, June 13, 2025. 约 15 倍/约 4 倍 token 倍数、90.2% 内部评测结果、半导体短缺的委派失败、CitationAgent 设计，以及按工作量扩展的规则都出自这里。https://www.anthropic.com/engineering/multi-agent-research-system
[^2]: "Uncontrolled Sub-Agent Recursive Loop Caused ~800k Token Consumption & Unexpected Charge," anthropics/claude-code, GitHub issue #69578, June 2026. https://github.com/anthropics/claude-code/issues/69578
[^3]: Anthropic, "Building Effective Agents," Anthropic Engineering, December 19, 2024. https://www.anthropic.com/engineering/building-effective-agents
[^4]: Walden Yan, "Don't Build Multi-Agents," Cognition blog, June 2025. https://cognition.ai/blog/dont-build-multi-agents
[^5]: Cognition, "Multi-Agent Systems in Production: Code Generation and Review at Scale," April 2026; 生产细节（Devin Review 平均每 PR 2 个 bug、58% 的严重占比、干净上下文评审者设计）以 ZenML LLMOps Database 同名条目中的摘要为准。https://www.zenml.io/llmops-database/multi-agent-systems-in-production-code-generation-and-review-at-scale
[^6]: Mert Cemri et al., "Why Do Multi-Agent LLM Systems Fail?", NeurIPS 2025; MAST 分类法，以及覆盖 7 个框架、1,600 条以上标注轨迹的 MAST-Data 语料库。arXiv:2503.13657. https://arxiv.org/abs/2503.13657
[^7]: The Linux Foundation, "A2A Protocol Surpasses 150 Organizations, Lands in Major Cloud Platforms, and Sees Enterprise Production Use in First Year," press release, April 2026; Google, "Announcing the Agent2Agent Protocol (A2A)," Google for Developers blog, April 9, 2025. https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year
[^8]: A2A Project, "Agent2Agent (A2A) Protocol Specification," v1.0, and the "A2A and MCP" topic page, Linux Foundation, accessed August 2026. https://a2a-protocol.org/latest/specification/
