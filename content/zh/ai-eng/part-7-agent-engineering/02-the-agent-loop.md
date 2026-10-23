---
title: "Agent 循环"
description: "思考、行动、观察，以及那个没人愿意谈的部分：循环怎么结束。停止条件、真正有效的反思，还有为什么生产环境的 agent 跟演示里完全是两回事。"
slug: 02-the-agent-loop
part: 7
chapter: 2
difficulty: intermediate
prerequisites:
  - 00-workflow-vs-agent
  - 01-the-five-workflow-patterns
  - part-2-working-with-model-apis/03-function-and-tool-calling
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# Agent 循环

有个团队（名字我就不说了）在周五上线了一个自主编码 agent。到周日早上，它已经花掉 4,200 美元，把同一个坏掉的工具调用了 8,100 次。bug 小得可怜：工具返回的是一个错误字符串，而 agent 的解析器默默把它当成了成功，于是模型"再试一次"，拿到同样的错误，再重试，循环永远关不上。没人设步数上限，没人设预算上限。模型手里有一个 `Finish` 动作，却始终走不到那一步——因为在它看来，活儿还没干完。

这个故事就是本章。来自 [ReAct 论文](../part-3-prompt-engineering/02-chain-of-thought-react-self-consistency.md) 的 think-act-observe 模式是所有人最先学的那部分，也是简单的部分。难的那部分——决定你的 agent 是一个产品还是一个烧钱无底洞的那部分——是循环怎么结束。生产环境的 agent 并不比研究演示更精巧，它们反而简单得多，只有一个大差别：一层厚厚的护栏，模型说服不了、也绕不过去。

## 循环到底是什么

把 [Function and tool calling](../part-2-working-with-model-apis/03-function-and-tool-calling.md) 里的函数调用往返跑不止一次，这就是 agent 循环（agent loop）。模型提出一个动作，你的 harness 执行它，你把结果追加进对话，然后再调一次模型。模型读自己之前的思考、做过的动作、回来的观察结果，再决定下一步做什么。

标准表述来自 Yao 等人 2023 年在 ICLR 发表的 ReAct 论文。[^1] 每次迭代由三个依次相连的部分组成：

- **Thought（思考）：** 模型写给自己的自由文本推理，从不发给任何工具。
- **Action（行动）：** 一次结构化输出，指名一个工具和它的输入。harness 解析它并执行。
- **Observation（观察）：** 工具的返回值，追加进上下文，供下一轮使用。

Thought 和 Action 出自同一次模型生成；Observation 是你的代码在跑完工具之后注入的。等模型下一次开口，它读到的是自己过去的思考、过去的动作，以及世界对它们的回应。这正是轨迹中途纠错之所以可能的原因：模型能看见自己错在哪，然后换一条路，就像你一条 `git status` 发现暂存错了文件那样。

![一次 ReAct 迭代的示意图：左边是模型，中间是工具执行，一条弧形箭头把观察结果送回下一轮的上下文。](../../../ai-eng/part-7-agent-engineering/images/02-react-loop-anatomy.png)
*循环的一次迭代：模型一起写出 Thought 和 Action；harness 运行工具并注入 Observation；下一轮读取此前发生的一切。*

一个最简 Python 骨架，砍到只剩必要的东西，二十行就够：

```python
def react_loop(llm_call, tools: dict, max_steps: int = 10) -> str:
    """Run the agent loop until the model finishes or max_steps trips."""
    messages = []
    for _ in range(max_steps):
        response = llm_call(messages)
        action = response.get("action", "")

        if action == "Finish":
            return response.get("action_input", "")

        tool_fn = tools.get(action)
        if tool_fn is None:
            observation = f"Error: unknown tool '{action}'"
        else:
            observation = str(tool_fn(response.get("action_input", "")))

        messages.append({
            "thought": response.get("thought", ""),
            "action": action,
            "observation": observation,
        })

    raise RuntimeError(f"Agent did not finish within {max_steps} steps")
```

就这些。二十行。现在的厂商把这一套包进了自家的 function calling API，你很少需要自己写解析器，但形状没变。演示和生产之间变化的，是它周围的一切。

## 停止条件才是真正的难题

有一个误解值得最先纠正：模型的 `Finish` 动作不是你的停止条件，它只是你停止条件的一个输入。模型会幻觉出"完成了"，就像它会幻觉出一条引用。生产事故里有一类被记在"当你的 agent 说完成了，其实什么都没做"这样的标题下：agent 信心十足地发出"任务完成"，而文件根本没写、webhook 根本没触发、数据库里那行一点没变。[^2] 模型在训练时见过太多"Done!"，不管活儿干没干，它都会写上一句。

所以真正的架构是两层。模型对"自己完没完"有它自己的看法；harness 有一套模型无法覆盖的独立限制。两者冲突时，harness 赢。

Anthropic Claude Agent SDK 暴露了两个由运营方控制的硬上限：`max_turns` 和 `max_budget_usd`，而且两者都没有默认值，你必须显式设置。[^3] Claude Code 的生产循环（2026 年 3 月通过 v2.1.88 的 source map 分析拆解出来）只返回五种带类型的退出原因：`success`、`error_max_turns`、`error_max_budget_usd`、`error_during_execution`、`error_max_structured_output_retries`。五种里有四种是运营侧停机，不是模型侧停机。[^4]

停止条件会以四种各不相同的方式失效，每一种你都该叫得出名字：

- **根本没有硬上限。** 循环一直跑到上下文耗尽，或者有人注意到账单。有一份 LLM agent 预算超支的案例汇编，记录了 63 起缺少进程内预算跟踪的事故。[^5]
- **死亡循环（loop of death）。** 模型用同样的参数调同一个工具，拿到同样的错误，再重试。它不是无限的，预算上限一触发就停——但到那时你已经花掉几千美元。
- **幻觉完成。** 模型发出 `Finish`，副作用却根本没发生。没有外部校验器去查数据库、查文件、查 HTTP 响应码，你就把错的结果悄悄发了出去。[^2]
- **上下文饱和。** 每一轮都把累积的全部历史重新喂一遍。工具输出超过几千 token 之后，信噪比就崩了，质量、延迟、成本一起恶化。[^4]

防御手段，按可靠程度从高到低排：

1. **外部校验器。** 跑测试套件、查数据库、看 HTTP 状态码。这是唯一无法被幻觉出来的停止条件，因为写它的不是模型。
2. **运营侧硬上限。** `max_turns`、`max_budget_usd`，再加一个挂钟超时。三个独立维度，全部由你的代码执行，提示词注入一个也碰不到。
3. **循环检测。** 对 (tool_name, input) 做滚动窗口哈希。五步之内出现两次同样的组合，直接中止。这比预算更早抓到死亡循环。
4. **模型自己的 `Finish` 动作。** 当提示有用，绝不能单独用。

![示意图：模型外面套着两层同心结构。内层是模型，带一个小小的 Finish 芯片；外层是运营方的 harness，三条标好名字的停止限制从外面喂进来。](../../../ai-eng/part-7-agent-engineering/images/02-stop-condition-layers.png)
*两层停止机制。模型的"Finish"住在里面；运营方的硬上限在外面，打平时永远它赢。*

harness 这一侧，写成代码比它挡下的故障模式还短：

```python
import time
from dataclasses import dataclass, field

@dataclass
class StopGuard:
    max_steps: int = 20
    max_wall_seconds: float = 300.0
    max_cost_usd: float = 5.0
    _steps: int = field(default=0, init=False)
    _start: float = field(default_factory=time.monotonic, init=False)
    _cost: float = field(default=0.0, init=False)

    def tick(self, step_cost_usd: float = 0.0) -> None:
        self._steps += 1
        self._cost += step_cost_usd
        elapsed = time.monotonic() - self._start
        if self._steps > self.max_steps:
            raise RuntimeError(f"exceeded max_steps={self.max_steps}")
        if elapsed > self.max_wall_seconds:
            raise RuntimeError(f"exceeded wall time {elapsed:.1f}s")
        if self._cost > self.max_cost_usd:
            raise RuntimeError(f"exceeded budget ${self._cost:.4f}")
```

每次循环迭代调用一次 `tick()`，在决定要不要继续之前调用。三个独立维度，全都在模型够不到的地方执行。`max_turns=20` 这个默认值只是起点；值得背下来的规则是设成你预期步数的两倍左右，再配一个美元上限。你以为五步能干完的任务，就设 `max_steps=10`。预算这个话题在[长时间运行的 agent](../../../ai-eng/part-7-agent-engineering/08-long-running-agents.md) 里展开，但每一次 agent 调用至少都要有这三行。

## 只有当模型之外的东西来验收时，反思才有用

"让模型检查自己的作业"听起来像一次免费升级，每个团队在准确率不达预期时第一个试的就是它。Huang 等人在 ICLR 2024 把这个实验认真做了一遍，跨多个推理基准得出的结论恰恰相反：让 LLM 在没有外部反馈的情况下重读并修改自己的答案，性能经常**下降**。[^6] 产生第一个答案的那个分布，同样产生了"这个答案没问题"的判断。模型改写了一遍，什么也没学到。

那反思什么时候真有用？当它锚在模型没写过的东西上时。Reflexion 是这个思路的标准形式化（Shinn 等人，NeurIPS 2023），它在内层 agent 循环外面套了一个外层试错循环，比 ReAct 多三个部件。[^7] **Actor** 跑一条轨迹。**Evaluator** 给它打分，而这是承重的那个词：Evaluator 是外部信号，是测试运行器、环境奖励、HTTP 响应码，不是模型的信心。试错失败，**Self-Reflection** 这一步写一份口头复盘（比如"I should have looked for the desklamp first"），追加进一个小记忆缓冲区。下一次试错开始前，先读累积下来的反思。

接地（grounding）是真的，数字才是真的。Reflexion 在 HumanEval Python 上拿到 91% pass@1，GPT-4 基线是 80.1%；ALFWorld 上绝对提升 22%，HotpotQA 上 20%，数据均截至 2023 年 10 月。[^7] 让 HumanEval 那个结果成立的关键是自生成单元测试：模型自己写测试，拿测试跑自己的代码，把失败当作 evaluator。测试写得不完美，但它们在实现之前就写好了，而且 Python 解释器不会在"测试过没过"这件事上撒谎。

最小可用的 Reflexion 外壳：

```python
from dataclasses import dataclass, field

@dataclass
class ReflexionMemory:
    max_experiences: int = 3  # Shinn 等人把记忆限制在 1-3 条，以适配上下文
    _experiences: list = field(default_factory=list, init=False)

    def add(self, reflection: str) -> None:
        self._experiences.append(reflection)
        if len(self._experiences) > self.max_experiences:
            self._experiences.pop(0)

    def as_context(self) -> str:
        if not self._experiences:
            return ""
        return "Previous attempt reflections:\n" + "\n".join(
            f"- {r}" for r in self._experiences
        )

def reflexion_trial_loop(actor, evaluator, self_reflect, task, max_trials=5):
    """Outer trial loop. evaluator MUST be external, not the model itself."""
    memory = ReflexionMemory()
    for _ in range(max_trials):
        trajectory = actor(task, memory.as_context())
        if evaluator(trajectory):       # 外部信号：测试、API、环境
            return trajectory
        memory.add(self_reflect(trajectory, reward=False))
    return None
```

决定要不要加反思循环的三条规则：

- **任务的通过/失败语义是干净的。** 测试要么过要么不过，API 要么返 200 要么返 500，查询结果要么对上要么对不上。如果你没法用几行确定性代码写出 evaluator，那你就没有 grounding，你只是又调了一次 LLM，让它假扮成 evaluator。
- **多次试错是可接受的。** 延迟和成本会乘以试错次数。等聊天回复的用户忍不了五次尝试，夜里跑的批处理任务可以。
- **首次尝试的准确率明显低于目标。** 如果模型已经 95% 一次做对，反思的开销收不回来。Reflexion 在模型失败多于成功的任务上才发光。

Reflexion 自己的论文也记录了一个值得记住的失效场景。在 WebShop 上，agent 试了四次没有任何提升，产出的反思也没什么用。[^7] 当任务需要模型没法从训练分布里生成的搜索策略多样性时，对过去几次尝试做反思帮不上忙。生产上的信号是：三到五次试错之后 evaluator 分数还不动，就停。找人、换工具、扩大搜索范围。别继续为不动的试错付钱。

## 为什么生产环境的 agent 比演示更简单

看一场研究演示，agent 干十件事：逛网站、规划行程、订酒店。再看一份熬过一年真实用户的线上代码，它只干一件事，范围很窄，轨迹很短。这不是巧合，这是算术。

独立步骤串起来的流水线，成功率是按乘法衰减的。单步 95% 正确的 agent，走 N 步就是 0.95 的 N 次方。十步是 59.9%，二十步是 35.8%，五十步不到 8%。[^8] 想在二十步上拿到 90% 的端到端成功率，单步准确率得是 99.5%。目前没有哪个模型在开放式工具使用上能稳定做到这个水平，而且这个差距不会靠扩大模型规模补上。

所以生产上的动作是把 N 做小。Anthropic 2024 年 12 月的生产指南说得很直接："最成功的实现并没有用复杂框架或专用库，它们用的是简单、可组合的模式。"建议是从一次优化过的 LLM 调用开始，简单方案不够用了再加结构，把 agent 循环当成最后的升级手段，而不是默认选项。[^9] [上一章](../part-7-agent-engineering/01-the-five-workflow-patterns.md)的五个工作流模式讲的就是先该拿什么。

另一个生产上的动作是让每一步都可恢复。十步里第七步失败又没有检查点，你就得从头再来；第七步失败但 harness 能从存下的状态只重试这一步，你就把乘法故障变成了加法故障。Claude Code 的循环，据 2026 年 3 月对 v2.1.88 source map 的分析，是 1,400 多行 TypeScript。朴素版本只有八行。[^4] 差的这 1,392 行是：指数退避重试、四段式压缩（工具结果预算、裁剪、微压缩、自动压缩）、把会话持久化到 JSONL 以支持 `--resume`、带类型的退出原因、stop hook 覆盖，以及双模型分工——小模型做记账，推理模型做规划。这些行没有一行是为了更聪明，全都为了不出错。

一句话概括从演示到生产的这道悬崖：如果任务需要二十步，而错误又不能独立恢复，那么开发时 90% 的成功率在生产里就是接近零的可用性。把"90%"读成"距离上线还有 90%"的团队，交出去的东西通常一周内就被用户弃用。

能活下来的形态：

- 少数几个设计良好的工具，而不是二十个薄包装。工具设计是一门自己的手艺，见[工具设计](../part-7-agent-engineering/03-tool-design.md)。
- 任务范围明确，成功标准写清楚。"改进这个代码库"不是任务，"修好 `billing_test.py` 里那个失败的测试"才是。
- 一个外部校验器来判定活儿到底干完没有。测试、schema 校验、API 健康检查，而不是模型的口头保证。
- 凡是花钱的维度都要有硬上限：轮数、挂钟时间、美元。没有例外。
- 每一轮都留追踪，出问题的时候你能把轨迹读回来，看是哪一步撒了谎。做法见[追踪](../../../ai-eng/part-8-evaluation-and-observability/06-tracing.md)。

放到架构尺度上，[Agent 架构](../../hld/part-9-ai-ml-system-design/03-ai-agent-architectures.md) 讲的是白板视角：这些循环怎么组合成能扛真实流量、有故障域、有长时间运行 agent 运维面的系统。从编辑席上看，那还是同一件事，只是范围更小。你在模型周围划边界，设置模型够不到的硬上限，把轨迹控制得短到算术算得过来。

## 参考文献

[^1]: Yao, Shunyu et al., "ReAct: Synergizing Reasoning and Acting in Language Models," ICLR 2023. https://arxiv.org/abs/2210.03629

[^2]: tianpan.co, "When Your Agent Says Done and Means Nothing," April 23, 2026. https://tianpan.co/blog/2026-04-23-hallucinated-success-agent-false-completion

[^3]: Anthropic, "How the agent loop works," Claude Agent SDK Documentation, as of June 2026. https://code.claude.com/docs/en/agent-sdk/agent-loop

[^4]: Meiyappan, Lax, "Why Claude Code's Agent Loop Is Over 1,400 Lines," INTERNALS.md, June 3, 2026. https://internals.laxmena.com/p/why-claude-codes-agent-loop-is-over

[^5]: Khan, Sajjad, "Token Budgets: An Empirical Catalog of 63 LLM-Agent Budget-Overrun Incidents," 2026. https://huggingface.co/papers/2606.04056

[^6]: Huang, Jie et al., "Large Language Models Cannot Self-Correct Reasoning Yet," ICLR 2024. https://arxiv.org/abs/2310.01798

[^7]: Shinn, Noah et al., "Reflexion: Language Agents with Verbal Reinforcement Learning," NeurIPS 2023. https://arxiv.org/abs/2303.11366

[^8]: tianpan.co, "Why Your 95% Accurate Agent Fails 40% of the Time," April 2026. https://tianpan.co/blog/2026-04-20-compound-accuracy-multi-step-agent-pipelines

[^9]: Schluntz, Erik and Zhang, Barry (Anthropic), "Building Effective Agents," December 19, 2024. https://anthropic.com/research/building-effective-agents
