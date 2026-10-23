---
title: "评估 RAG 与 agent"
description: "组件评测、工具选择准确率、轨迹评估，以及“最弱一环”模式——它告诉你复合系统里该修哪一块。"
slug: 03-evaluating-rag-and-agents
part: 8
chapter: 3
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
  - part-6-retrieval-augmented-generation/07-evaluating-retrieval
  - part-7-agent-engineering/02-the-agent-loop
  - part-7-agent-engineering/03-tool-design
tags: [llm, observability, rag, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 评估 RAG 与 agent

2024 年 6 月，Sierra Research 公布了一个让所有 agent 团队都不舒服的数字。他们的 tau-bench 基准让 GPT-4o 对同一个零售客服任务跑了八次独立运行。任何单次运行，它大约有一半的时候能成功。八次跑完，只有四分之一的情况是每次都成功。对一个最终答案评测（final-answer eval）判定为通过的模型来说，可靠性掉了 60%。[^1]

agent 并没有在两次运行之间变差。是评测看错了地方。

端到端评测（end-to-end evaluation）是大多数团队上线时带着的东西：发一个查询，给最终答案打分，盯着看板。它便宜，它覆盖了用户的真实体验，而在复合系统上它会一直骗你。一个召回率 40% 的检索器喂给强模型，端到端答案质量照样能做到 80%，因为模型会用参数化知识把缺口填上。[^2] 一个 agent 绕 12 步、穿过三个死胡同才摸到正确答案，与另一个干净走 3 步的 agent 得分完全一样。总体质量一直是绿的；bug 一直隐身；你花两周调错了组件。

解法不是一个更好的端到端指标。解法是把各组件分开评，把系统当成一条链——它的质量受最弱一环限制。

## 最弱一环的调试模式

一条 RAG 流水线就是一条链：检索器、重排序器、生成器。agent 是一条更长的链：规划器、工具选择器、参数构造器、工具执行器、回答撰写器，有时还有一个回环的评审者。端到端评测问的是“最后一环有没有产出正确输出？”这是个有用的冒烟测试，但当答案是否的时候，它完全没告诉你哪一环断了。

组件评测把顺序倒过来。你给每个组件自己的指标，在自己的标注数据集上跑，然后从上往下读分数，直到找到红的那一个。接着只修它。流程是机械的：

1. 端到端评测标出一次回归。
2. 用一份固定的标注集跑一遍每个组件的评测。
3. 找出分数掉了、或者低于阈值的那个组件。
4. 修这个组件，只修这个组件。
5. 重跑该组件评测确认；再跑端到端，看有没有新的回归。

第 2 步的顺序有讲究。先跑便宜的确定性指标，最后跑贵的 LLM-as-judge 指标。对 RAG 系统来说，就是先跑检索指标（Recall@K、Precision@K、MRR），再跑生成指标（忠实度 faithfulness、答案相关性 answer relevance），因为检索指标是免费的，而且它们能抓住最常见的失败模式。[评估检索](../part-6-retrieval-augmented-generation/07-evaluating-retrieval.md) 详细讲了这些指标和“检索 vs 生成”的 2×2 矩阵；本章假设你已经把它们接好了，转而讨论 agent 这一侧——那里链更长，失败模式成倍增长。

Barnett 等人在 2024 年研究了三个生产环境的 RAG 系统，梳理出七个不同的失败点，从“文档根本没进索引”一直到“LLM 忽略了格式指令”。[^2] 其中三个在检索里（Recall@K 能发现），三个在生成里（faithfulness 能发现），还有一个落在两者的接缝上（某个分块被检索到了，但在重排序或提示词拼装时丢了）。没有组件评测，这七种从外面看一模一样：答案错了。

对 agent 来说，链更长，接缝更多。一个用工具的 agent 可能因为选错工具、工具选对了但参数传错、调用顺序错、陷入循环、撞上上下文上限，或者干脆放弃而失败。每一种都想要自己的指标。

## 三个指标，对应 agent 可能做错的三件事

用工具的 agent 有三层正确性，每一层你都需要一个指标，才能把它们区分开。

**工具调用准确率（tool-call accuracy）** 问的是 agent 有没有用正确的参数、按正确的顺序调用了正确的工具。Ragas 的 `ToolCallAccuracy` 接收一份期望工具调用列表作为参考，与 agent 实际调用出来的序列比对。严格模式（默认如此）下，分数等于参数匹配率乘以顺序匹配指示位：顺序错了分数直接归零，无论参数匹配了多少个。灵活模式（flexible mode）去掉顺序要求，对“取三条互不相关的记录”这类可并行安全的操作，这是正确的选择。[^3]

**工具调用 F1（tool-call F1）** 是同一件事，但按部分得分来评。它把期望调用的集合和实际调用的集合当作标签，统计真正例（工具对、参数也对）、假正例（多余的调用）和假反例（漏掉的调用）。早期迭代、想看到进展时用 F1；流程必须精确时用准确率，比如一个必须先读再写的数据库迁移 agent。[^3]

**Agent 目标达成度（agent goal accuracy）** 完全不看路径，只问一个是非题：agent 有没有做到用户想要的事？由 LLM judge 拿对话的最终状态与参考结果比对，返回 0 或 1。当用户不关心调了哪些 API、只要预订成功就行时，用它。[^3]

这三个是互补的，不是冗余的。一个预订 agent 可以在目标达成度上拿 1.0（“订单确实存在”），同时在工具调用 F1 上只拿 0.5，因为它先试了两个错的 API。这是诊断金矿：流程低效，但结果成立。反过来更危险，也更常见，下一节就讲这个。

下面是用 Ragas 搭的最小评测脚手架：

```python
from ragas.metrics.collections import ToolCallAccuracy, AgentGoalAccuracyWithReference
from ragas.llms import llm_factory
from openai import AsyncOpenAI

async def eval_agent(conversation, reference_tool_calls, reference_goal):
    llm = llm_factory("gpt-4o-mini", client=AsyncOpenAI())
    tool_acc = ToolCallAccuracy()  # 默认按严格顺序
    goal_acc = AgentGoalAccuracyWithReference(llm=llm)

    tool_score = await tool_acc.ascore(
        user_input=conversation,
        reference_tool_calls=reference_tool_calls,
    )
    goal_score = await goal_acc.ascore(
        user_input=conversation,
        reference=reference_goal,
    )
    return {
        "tool_call_accuracy": tool_score.value,
        "agent_goal_accuracy": goal_score.value,
    }
```

`ToolCallAccuracy` 是确定性的，除了标注之外不花钱。`AgentGoalAccuracyWithReference` 会调 LLM judge，用 `gpt-4o-mini` 大约每条样本半分钱。把便宜的那个放进 CI，每个 PR 都跑；judge 留给夜间批量跑。

## 轨迹评估（trajectory）：重要的是路径，不只是终点

两个 agent 可以用截然不同的计划产出完全相同的最终答案。一个读完查询，选对工具，返回答案。另一个试了三个错的工具，搞糊涂了，绕了一圈，恢复过来，最后撞进同一个答案。目标达成度给它们同样的分数。工具调用准确率能抓住这个差别，但前提是你有一条参考轨迹可以比对。

![两条 agent 轨迹抵达同一个答案：一条是干净的三步路径，另一条是穿过死胡同和一次循环的十二条缠绕弯路，两者都通过最终答案评测，但只有干净的那条通过轨迹评测](../../../ai-eng/part-8-evaluation-and-observability/images/03-trajectory-vs-final-answer.png)
*最终答案评测分辨不出一份靠谱的计划和一个靠运气成功的脆弱计划；轨迹评测可以。*

这不是学术层面的担心。脆弱的那条路径在你测的这一跑里有效，只因为用户的措辞恰好把 agent 往恢复的方向推了一把。提示词里改一个词，改一个工具的响应时间，下周二再跑一遍，同一个 agent 就会一直转圈。

LangSmith 把轨迹评估拆成三条彼此嵌套的轴：[^4]

- **最终回复（final response）。** 最后一条消息里有没有正确答案？这是最便宜的检查，也是大多数团队已经有的那个。
- **单步（single step）。** 取轨迹里的任意一步，孤立地打分：考虑到那一刻已知的信息，这次 LLM 调用的工具选择对不对？定位问题出在哪很好使。
- **完整轨迹（full trajectory）。** 拿整条序列与参考比对。严格版本是完全匹配（同样的工具、同样的顺序）；宽松版本是集合匹配（同样的工具、顺序任意）；没有硬性参考时，用 LLM judge 通览整条 trace 的弹性版本。

成本阶梯从免费（最终回复，确定性）一路走到昂贵（对整条 trace 跑 LLM judge，常常每条样本两到五分钱）。参考数据的采集成本同向增长：写下“正确答案是 X”很便宜；写下“正确的计划是先 call_tool_A 再 call_tool_B、参数是这些，如果失败了就退回到……”很贵。大多数团队从最终回复开始，需要 debug 时加工具调用 F1，只有可靠性变成硬要求时才上完整轨迹评测。

## 当 pass@1 说谎时：pass^k 与可靠性

大多数 agent 基准报 pass@1：每个任务跑一次，数成功。这是乐观指标，是让 demo 好看的那个。悲观指标是 pass^k：每个任务独立跑 k 次，数“每一次都成功”的情况。可靠性就活在这两者的差距里。

这个差距大得离谱。tau-bench 在零售客服任务上测 GPT-4o：pass@1 约 50%，pass^8 约 25%。[^1] Sierra 在 2024 年中测的十二个模型里，每一个都随 k 增大而大幅下滑。Anthropic 的 Claude 3.5 Sonnet 在 pass@1 上是最强的选手，到 pass^8 时掉进了和最弱模型差不多的区间。[^1] 即便能力很强，可靠性也不是白给的。

机制很简单。一个在独立尝试上成功率 80% 的 agent，八次全部成功的概率是 0.8^8，也就是 17%。如果你的产品每分钟有八个不同用户问同一类问题，这才是你的工单系统在追踪的指标。pass@1 是你的 demo 在测的东西。

评测里一个实用的 pass^k 循环：

```python
async def pass_k_eval(agent, task, k=8):
    """Run the same task k independent times; report both metrics."""
    successes = 0
    for _ in range(k):
        result = await agent.run(task.input, fresh_state=True)
        if task.check(result):
            successes += 1
    return {
        "pass_at_k": 1 if successes >= 1 else 0,
        "pass_pow_k": 1 if successes == k else 0,
        "success_rate": successes / k,
    }
```

`"fresh_state=True"` 这一点很关键：每一次运行都必须独立（新对话，不携带记忆），否则这个指标就退化成了伪装过的 pass@1。评测集上 pass^k 的平均值，才是你真正想追踪的生产可靠性数字。AgentBench 在 ICLR 2024 上跨八个环境评测了 29 个 LLM，发现生产环境的主导失败模式不是“答案错了”，而是“卡在循环里”：超过 90% 的超时轨迹中，最后十轮里至少有两轮回答的字面重合度很高。[^5] 一个靠第一次尝试撞对而通过 pass@1 的 agent，早晚会有撞不对的一天然后开始转圈，pass^k 就是那个在用户之前把这种风险暴露出来的指标。

组件评测告诉你哪一环最弱。pass^k 告诉你连拉八次这条链还撑不撑得住。两个都不上，你就是靠 Sierra 的数据已经证明会骗你两倍的那个指标在飞。

在架构层面，[AI 系统的评测与可观测性](../../hld/part-9-ai-ml-system-design/05-llm-evaluation-observability.md) 讲了怎么把这些组件指标接进线上的真实流量，配上 trace、看板和告警。

## 参考文献

[^1]: Yao, Shinn, Razavi, Narasimhan, "tau-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains," arXiv:2406.12045, June 2024. https://arxiv.org/abs/2406.12045
[^2]: Barnett, Kurniawan, Thudumu, Brannelly, Abdelrazek, "Seven Failure Points When Engineering a Retrieval Augmented Generation System," 3rd International Conference on AI Engineering, April 2024. https://arxiv.org/html/2401.05856v1
[^3]: Ragas documentation, "Agentic or Tool use metrics," docs.ragas.io, December 2025. https://docs.ragas.io/en/latest/concepts/metrics/available_metrics/agents/
[^4]: LangChain, "Application-specific evaluation approaches," LangSmith documentation, 2025. https://docs.langchain.com/langsmith/evaluation-approaches
[^5]: Liu et al., "AgentBench: Evaluating LLMs as Agents," ICLR 2024, Tsinghua University, arXiv:2308.03688. https://arxiv.org/html/2308.03688v3
