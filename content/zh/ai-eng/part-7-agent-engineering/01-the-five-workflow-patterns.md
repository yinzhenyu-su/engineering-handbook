---
title: "五种工作流模式"
description: "链式调用、路由、并行化、编排者-工作者、评估者-优化者：每一种都配了代码、一个用例，以及它在生产环境里真正咬人的失效模式。"
slug: 01-the-five-workflow-patterns
part: 7
chapter: 1
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/03-function-and-tool-calling
  - part-2-working-with-model-apis/04-errors-retries-fallbacks
  - part-3-prompt-engineering/03-templates-and-chaining
  - 00-workflow-vs-agent
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 五种工作流模式

2024 年 12 月，Anthropic 应用团队发布《Building effective agents》，对 agent 框架行业杀伤力最大的就是这一句：“最成功的实现并没有用复杂框架……它们用的是简单、可组合的模式。”[^1] 文章随后给这些模式起了名字。五个。你听说过的每一个生产环境“agent”，都是这五个原语的某种组合，再加上工具。

本章逐个讲这些模式：让它落地的代码、它真正适用的任务，以及它出错的那种具体方式。模式按复杂度递增排列，这个顺序是有实际作用的：明明轻量模式够用却上更重的模式，是这个领域里最常见、也最贵的错误。[^1] 给它们起名字的全部意义，是让“次便宜的那个选项”变得可见，这样你就可以停在它那儿。

![五种工作流模式分布在两个维度上：工作如何分解，以及如何执行。](../../../ai-eng/part-7-agent-engineering/images/01-five-patterns-map.png)
*左边是静态分解，右边是动态分解；上方是顺序执行，下方是并行执行。评估者-优化者多了一层循环。越往右走、越加迭代，模式越贵。*

## 1. 链式调用（chaining）

一条链是固定的 LLM 调用序列，每一步的输出就是下一步的输入。步骤要提前写死。

经典形态大致是：先列文档大纲，再按标准检查这份大纲，然后依据通过的大纲写正文。[^1] 每次调用只干一件事，比一次干四件事容易。

```python
from typing import Callable

def chain(input_text: str, steps: list[Callable[[str], str]]) -> str:
    """Run steps sequentially, gating between each one."""
    result = input_text
    for step_fn in steps:
        result = step_fn(result)
        # 闸门检查放在这里：解析、校验，输出不合格就中止。
    return result
```

这笔交易很实在：四步的链，延迟至少是单次调用的 4 倍，而且每一步都在烧 token。换回来两样东西。每次调用都是更简单的任务，模型处理得更好；第 4 步产出垃圾时，你能翻回去看第 2 步到底吐了什么。

失效模式是**级联错误传播（cascading error propagation）**，这也是链在生产环境里静默失败最常见的原因。AgentEval 对 450 条生产 agent 轨迹的分析（2026 年 4 月）发现，63% 的步骤级失败并非本步引起，而是上游错误被下一步当成事实接受了。[^2] 上下文丢失的放大效应最严重，下游放大 3.2 倍。[^2] 机制很残忍：LLM 不会因为输入有问题就抛异常。它会产出一段自信、合理的输出，把混进来的那个畸形东西一起吸收掉。到第 4 步，最初的错误已经被三轮流畅的文字洗白了。

修法就是闸门检查，上面代码里那行注释。每步之间，解析上一步的输出，断言它格式合规，下一步才允许看到它。端到端的结果检查（最终答案看着对不对？）只能抓住步骤级检查能抓住的 41% 的失败。[^2] 中间输出也要记日志。凌晨两点生产出事，你要问的问题是“那是哪一步产出的？”，唯一可接受的回答是一条能告诉你的 trace。

[模板与链式调用](../part-3-prompt-engineering/03-templates-and-chaining.md) 讲的是同一个原语在提示词模板那一侧的做法。

## 2. 路由（routing）

路由器先对输入分类，再把它派给专门的处理器。分类器通常是一次小模型的 LLM 调用；处理器可以是不同的提示词、不同的模型，或者不同的工具集。

生产里最常见的用法是模型档位路由：简单问题走便宜模型（Haiku 4.5），难问题走强模型（Sonnet 4.5）。[^1] 客服工单同理：退款走退款提示词加退款工具，账单问题走账单提示词，只带对发票服务的只读权限。

```python
from typing import Callable

def route(
    user_input: str,
    classify_fn: Callable[[str, list[str]], str],
    handlers: dict[str, Callable[[str], str]],
    fallback: str = "default",
) -> str:
    """Classify, then dispatch to the matching handler."""
    chosen = classify_fn(user_input, list(handlers.keys())).strip().lower()
    handler = handlers.get(chosen) or handlers.get(fallback)
    if handler is None:
        raise ValueError(f"No handler for '{chosen}' and no fallback.")
    return handler(user_input)
```

路由要值回票价，得两个条件同时成立：输入能分成若干类别，且各类的最优处理方式确实不同；分类器在你的真实流量上可靠。两半都重要。95% 准确率、省下 60% 推理成本的路由器是赚的；75% 准确率、把四分之一客户派给错专人的路由器，还不如没有路由器。

失效模式是**路由失准（router miscalibration）**，而且比听起来更糟，因为它不可见。Tian Pan 记录过一个 SaaS 部署：目标是 30% 的请求升级给昂贵模型；分类器的决策边界偏了，实际升级了 60%。[^3] 被“就地”处理的那 40% 查询给出的答案更差，用户重试更多，总请求量上升，推理账单反而涨了 12%——尽管路由器做的事正是它被造出来要做的事：路由。遥测显示路由工作正常。质量成本和重试率不在路由器的看板上。

有两件事确实管用。**拿真实流量做校准**，别用合成样本；用户真正会问什么的分布，是唯一重要的分布。**别天真地信任分类器的置信度分数**：GPT-4o-mini 的误分类集中在高置信度那一端，大约三分之二的错误发生在模型报出 >80% 置信度的时候。[^4] 靠置信度阈值做的兜底（“没把握就升级”）恰好会在最需要它的那些 case 上失效开放。

## 3. 并行化（parallelization）

并发跑多个 LLM 调用，再合并结果。它有两个变种，把两者混为一谈，是烂设计一个不大却长期存在的来源。

**切分（sectioning）** 把任务拆成互相独立的子任务。要分析一项政策变动对客户、员工、投资人和供应商的影响？发四次独立调用，每个利益相关方一次，并行跑。每次调用里，模型的注意力全在一个视角上。Anthropic 的护栏模式是同一个思路：一次调用回答用户，另一次同时筛查请求是否违反政策，而不是把两件事挤进一个提示词让它一次干完。[^1]

**投票（voting）** 把*同一个*任务跑 N 次再汇总。比如代码安全审查：三个独立提示词各自找漏洞，一条发现要至少两票同意才留下。你用 N 倍成本换更低的误报率。

```python
from concurrent.futures import ThreadPoolExecutor
from typing import Callable

def parallel(
    prompt_fn: Callable[[str], str],
    inputs: list[str],
    n_workers: int = 4,
) -> list[str]:
    """Run prompt_fn over independent inputs concurrently."""
    with ThreadPoolExecutor(max_workers=n_workers) as executor:
        futures = [executor.submit(prompt_fn, x) for x in inputs]
        return [f.result() for f in futures]
```

子任务确实独立（A 的输出从不做 B 的输入）、而且你在意延迟时，用切分：并行执行的延迟接近最慢那一次调用，而不是它们之和。一次错答案贵到值得付 N 倍成本去压低误报率时，用投票。

失效模式是**汇总逻辑**。并行输出必须被综合，而综合这一步一草率，各 worker 之间的矛盾就会被抹平成一句自信的错误结论。worker A 假设用户是免费档，worker B 假设是付费档；综合步骤把差异折中，推荐了一条根本不适用的升级流程。修法是给综合步骤一条显式的矛盾检测提示词：“如果各输入在任何事实前提上不一致，指出来并停下。”

投票有同一个问题的自家版本：投票阈值设计。要求全体一致，你会漏掉真实发现（假阴性）；任意一票就算，你等于花 3 倍成本跑了一次调用（假阳性）。阈值是在你的 eval 集上做的校准问题，不是凭直觉。上面那个朴素的 `ThreadPoolExecutor` 也没有按 worker 划错误边界，一次超时就能毒化整批；生产代码会把每个 future 包起来，把部分失败当数据，而不是当异常。

## 4. 编排者-工作者（orchestrator-workers）

一个中心编排者（orchestrator）LLM 看输入，决定需要哪些子任务，把每个子任务派给一个 worker LLM。worker 交回结果，编排者做综合。

它和并行化的区别在动态扇出。静态并行永远发 4 个利益相关方的调用，因为你在代码里写了“四个利益相关方”。编排者-工作者可能判定*这个*编码任务涉及三个文件，就开三个 worker；下一个涉及十一个文件，就开十一个；再下一个只需要一处精准修改，就开一个。[^1] 分解是在运行时、由 LLM、针对具体输入决定的。

典型的用例是编码 agent（该改哪些文件取决于 bug）和研究 agent（该查哪些来源取决于问题）。[^1] Anthropic 那个能在 SWE-bench 上从 PR 描述解决真实 GitHub issue 的编码 agent，跑的就是这个模式。[^1]

失效模式是**编排者规划失败**，它是所有下游 worker 问题的母体。编排者理解错了任务，或者生成的子任务描述不清，每个 worker 都会忠实执行，合起来的结果就是错的或不全的。AgentEval 的失败分类里，目标误解占 agent 失败的 12%，漏步骤占 8%，顺序错误占 5%，全都在任何 worker 的上游。[^2] 有一个具体亚种：编排者吐出的 XML 或 JSON 调度器解析不了，整件任务在一个 worker 还没跑起来之前就失败了。

有用的纪律是让规划这一步可检查。提示编排者说出覆盖理由（“每个子任务对应任务的哪个方面？”），并把这份理由当闸门用。派发前先校验结构化输出。还要记住它的成本形状：编排者-工作者最少 N+1 次 API 调用（1 次规划 + N 个 worker），是每种任务里最贵的模式。[^5] 只有当分解确实依赖输入时才用它。如果你在看到输入之前就能把分解写下来，就用静态并行，更便宜也更好调。

放到架构尺度，这就是协调者-工作者（coordinator-worker）分布式模式在 AI 系统里的实例，单点故障的性质也一样；HLD 第 9 部分里 [AI Agent 架构](../../hld/part-9-ai-ml-system-design/03-ai-agent-architectures.md) 那一章讲的是白板上的视角。

## 5. 评估者-优化者（evaluator-optimizer）

一个 LLM 出草稿。第二个 LLM 评估它并给反馈。第一个带着反馈重写。循环到评估者打出 PASS，然后返回。

这个模式适用于成功标准清晰、可表述，而且初稿和打磨后的成品之间差距有意义的任务：文学翻译、复杂检索，任何“先批评再修改”正好是人的做法的活。[^1] 但在写代码之前你得先读完失效模式，因为这是最可能无声烧光你 token 预算的那个模式。

危险的失效模式是**验证者-生成者盲区**。同一个模型（或同一家族）同时扮演两个角色时，评估者会继承生成者的偏见，包括它的错误。2025 年一项覆盖 14 个模型的基准发现，模型在别人输出里本来会标出来的错误，有 64.5% 在自己评估自己时漏掉了。[^6] 机制很直白：两次调用抽自同一个底层分布，一段自信的错误生成，在同家族的评估者眼里就是自信的正确。

生产里长这样：Tian Pan 记录过一条轨迹，循环迭代了 12 次，评估分数在 0.78 到 0.84 之间徘徊，始终没过 0.90 的 PASS 阈值，运行在 3 小时时超时，烧掉的 token 够付一个高级工程师一天工钱的四分之一。[^7] 每次迭代增加的是自信，不是信息。校准误差随迭代次数上升；模型变得更确定，而不是更正确。

成本形状会把它放大。一个 10 轮的 Reflexion 式循环，token 消耗大约是单次线性执行的 50 倍，因为每次迭代都要重放完整对话历史。[^7] 边际价值掉得很快：自我纠正能带来的改进，约 75% 落在头两轮。[^7] 第 12 轮不是在修答案，是在烧预算。

能在生产里活下来的版本，把三个特性焊在了里面。

```python
from typing import Callable

MAX_ITERATIONS = 3  # 自我纠正约 75% 的收益落在第 1-2 轮。

def evaluator_optimizer_loop(
    task: str,
    generate_fn: Callable[[str, str], str],         # (task, feedback) -> draft
    evaluate_fn: Callable[[str, str], tuple[str, str]],  # -> (status, feedback)
) -> tuple[str, list[str], bool]:
    """Bounded loop with explicit escalation when the budget is exhausted."""
    feedback = ""
    history: list[str] = []

    for _ in range(MAX_ITERATIONS):
        draft = generate_fn(task, feedback)
        history.append(draft)
        status, feedback = evaluate_fn(task, draft)
        if status == "PASS":
            return draft, history, True

    return history[-1], history, False  # 调用方必须处理这个 False
```

第一，**硬的迭代上限，由编排者设定，循环本身无权改**。对多数任务来说 3 是合理的默认值。第二，**异构验证者**。绕开盲区最便宜的办法是非 LLM 检查：代码用测试运行器，结构化输出用 JSON schema 校验器，查询用 SQL 解析器。这些做不到时，让评估者用与生成者不同的模型家族。第三，**显式的升级路径**，用于循环没通过就退出的时候，也就是上面那个 `False` 返回值。由编排者决定接下来怎么办：换更强的模型重试、转给人工，或者把最好的一次尝试连同信心标记一起交出去。悄悄把最后一版草稿当成通过了返回，就是这个模式名声变坏的来由。

如果你盯着评估分数随迭代的变化，看到它在一个窄带里低于阈值来回震荡，那循环是卡住了，不是在收敛。杀掉它。

## 怎么在它们之间选

默认答案是一次 LLM 调用。配了检索、给了好示例的优化提示词，能用这里最便宜模式五分之一的成本和五分之一的延迟，处理掉大多数任务。只有当单次调用在你的 eval 集上被证明失败时，才去动这些模式。[^1] 真要去动，顺序很重要：

| 如果你的任务…… | 用 |
|---|---|
| 有可以提前写下来的串行子目标 | 链式调用 |
| 能分成已知类别，各类的最优处理方式不同 | 路由 |
| 有你在事前就知道的独立子任务（或者高风险判断需要 N 票一致） | 并行化 |
| 需要依赖具体输入做分解 | 编排者-工作者 |
| 有清晰的通过/不通过标准，且能从“先批评再修改”中获益 | 评估者-优化者（配异构验证者和硬上限） |

这些模式可以组合。路由器可以按查询类型把请求派给一条链，或者一个编排者-工作者子图。一个评估者-优化者循环可以塞在编排者的某个 worker 里。Anthropic 对组合的建议和对模式本身的一样：只有在测到了改进时才加复杂度。[^1] 2026 年交付最可靠 AI 产品的那些团队，不是靠叠加模式走到那儿的；他们是停在第一个跑通的模式上。

## 参考文献

[^1]: Erik Schluntz、Barry Zhang，“Building effective agents”，Anthropic，2024 年 12 月 19 日，https://www.anthropic.com/research/building-effective-agents
[^2]: Dongxin Guo、Jikun Wu、Siu Ming Yiu，“AgentEval: DAG-Structured Step-Level Evaluation for Agentic Workflows with Error Propagation Tracking”，arXiv:2604.23581v1，2026 年 4 月 26 日，https://arxiv.org/html/2604.23581v1
[^3]: Tian Pan，“Model Routing in Production: When the Router Costs More Than It Saves”，tianpan.co，2026 年 4 月 18 日，https://tianpan.co/blog/2026-04-18-model-routing-production-when-router-costs-more
[^4]: Tian Pan，“Why Accuracy Is the Wrong Metric: LLM Classifiers in Production Beyond Accuracy”，tianpan.co，2026 年 5 月 4 日，https://tianpan.co/blog/2026-05-04-llm-classifier-production-beyond-accuracy
[^5]: Anthropic，“Orchestrator workers”（cookbook），2024 年 12 月，https://platform.claude.com/cookbook/patterns-agents-orchestrator-workers
[^6]: 转引自 Tian Pan，“The Self-Correction Loop That Shared Its Verifier's Blind Spot”，tianpan.co，2026 年 6 月 2 日（2025 年一项覆盖 14 个模型的基准），https://tianpan.co/blog/2026-06-02-the-self-correction-loop-that-shared-its-verifiers-blind-spot
[^7]: Tian Pan，“The Self-Correction Loop That Shared Its Verifier's Blind Spot”，tianpan.co，2026 年 6 月 2 日，https://tianpan.co/blog/2026-06-02-the-self-correction-loop-that-shared-its-verifiers-blind-spot
