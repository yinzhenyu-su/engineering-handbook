---
title: "人在回路（Human-in-the-loop）"
description: "把审批门做成工具调用、按置信度升级、以及交接设计：要让有选择性的人工监督真正生效，这三件彼此独立的事你都得自己做。"
slug: 07-human-in-the-loop
part: 7
chapter: 7
difficulty: intermediate
prerequisites:
  - 00-workflow-vs-agent
  - 02-the-agent-loop
  - 03-tool-design
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 人在回路（Human-in-the-loop）

Anthropic 发布 Claude Code 时，给每一个有实质后果的工具调用都上了一道权限确认提示。用户批准了其中的 93%。[^1] 这不叫监督，这叫给点击反射套了个确认对话框。

人在回路（human-in-the-loop）的幼稚做法是“加个批准按钮”。生产做法是三个互相独立的工程问题，每一个都能单独把你的方案搞坏。*哪些*动作设门，决定了人到底有没有机会看到真正重要的东西；阈值*怎么*定，决定了 agent 会不会把该上报的调用上报上来；人在门里*看到*什么，决定了他是真做了判断，还是照例盖章。只优化其中一项，另外两项就会退化。

本章所有结论都压在一条经验信号上，它来自 Anthropic 对 2026 年初 50 万次 Claude Code 会话的分析。[^2] 新用户（不到约 10 次会话）大约 20% 的情况下开自动批准，并打断 5% 的轮次。老用户（750 次以上会话）自动批准的比例超过 40%，打断 9% 的轮次。他们不是变懒了，是监督得更好了。他们学会了：盯着 agent 跑、看着不对再插手，比对着每条 shell 命令条件反射地点批准强。

从逐动作审批转向“监控加打断”，这个转变就是设计目标。本章剩下的部分，是凑齐这个目标的三块积木。

## 审批门其实是被伪装起来的工具调用

主流框架都不约而同用了同一个招：设了门的工具并不是什么特殊的“暂停”原语，它就是一个普通工具调用，只不过运行时把它拦下来、序列化，再呈现给人看。人的决定作为工具结果返回。agent 循环根本不知道中间发生过什么异常。

正是这种同构让 HITL 可以组合。你的 agent 本来就会等工具调用、读结果、再调整。审批不过是一个实现体是“人”的工具。

具体到 OpenAI Agents SDK 里：

```python
from agents import function_tool

@function_tool(needs_approval=True)
async def cancel_order(order_id: int) -> str:
    """Always requires human approval before cancelling."""
    return f"Cancelled order {order_id}"


async def requires_review(_ctx, params: dict, _call_id: str) -> bool:
    """Per-call: escalate only when the email looks like a refund."""
    return "refund" in params.get("subject", "").lower()


@function_tool(needs_approval=requires_review)
async def send_email(subject: str, body: str) -> str:
    return f"Sent '{subject}'"
```

runner 碰到设门的调用时不会执行工具，而是把 agent 名、工具名和精确参数装进 `RunResult.interruptions` 里的一个 `ToolApprovalItem`，然后把控制权交回你的应用。你用 `result.to_state()` 把运行状态序列化出来，存到数据库、队列或文件里，几小时甚至几天后再用 `state.approve(interruption)` 或 `state.reject(interruption, rejection_message="...")` 恢复，接着调 `Runner.run(agent, state)`。[^3] 运行会从当初暂停的那一点接着往下走。

LangGraph 用命令式的形态实现了同一个模式：节点里的 `interrupt()` 调用抛出一个特殊异常，运行时把图状态存进配置好的 checkpointer，调用方用 `Command(resume=<value>)` 恢复。[^4] Anthropic 的 Claude Agent SDK 用一个 `canUseTool` 回调，返回 `PermissionResultAllow(updated_input=...)` 或 `PermissionResultDeny(message=...)`。[^5]

看明白之后，两个设计选择就自己浮出来了。

**拒绝理由是 agent 会读的反馈。** 人写下“不行，这会删掉生产数据，改走 staging 端点”，这段文字就成了模型下一轮看到的工具结果。agent 读到为什么被拒，然后调整。只给一个光秃秃的“rejected”，等于训练它要么重试、要么放弃；给一个真实解释，它才能纠偏。OpenAI 和 Anthropic 的 SDK 都允许你按调用自定义拒绝文案。[^3][^5]

**“带修改批准”是整套技术栈里最被低估的功能之一。** Claude Agent SDK 允许审批者在执行前改写工具的输入：`PermissionResultAllow(updated_input=modified_input)`。[^5] 模型看到的是改过之后的调用结果，不是原来那个。就这一个钩子，把人从二值守门员变成了编辑。批准一条数据库查询，但把它限定在单个租户内。批准一次文件写入，但把它重定向到沙箱路径。批准这封邮件，但把客户全名从标题里去掉。

> [!WARNING]
> **LangGraph 在恢复时会从头重跑你的节点。** 人每批准一次，`interrupt()` 之前执行过的代码就再跑一遍。如果你在门前写了 `db.create_audit_log(...)`，每次恢复都会多出一条审计日志。把副作用挪到 interrupt 之后，或者让所有 interrupt 之前的操作都幂等（用确定性 ID 做 upsert，而不是 insert）。[^4]

还有一个值得从兄弟章节借过来的设计模式：设门的工具对从外部 [MCP 服务器](04-mcp-model-context-protocol.md) 加载进来的工具同样有效。两个 SDK 都允许你把整个 MCP server 或特定的 MCP 工具标记为 `require_approval`，于是不用改第三方工具就能把 HITL 延伸过去。[^3][^5]

## 按置信度升级：阈值不是拍脑袋定的

门禁这套管线定下来之后，下一个问题是：哪些工具调用真的要走这道门。全设门，就是那个 93% 照例盖章的问题。一个都不设，就是 `--dangerously-skip-permissions` 干的事。中间路线是按 agent 对自己动作的置信度，有选择地上报。

背后有个干净的公式，出自 DosSantos DiSorbo 和 Ju 2026 年 3 月那篇关于语言模型升级行为的论文。[^6] 把这个决策当成一笔成本计算。设 `c_l` 为转给人工评审的成本，`c_w` 为 agent 做错事的成本。最优阈值是：

```python
from dataclasses import dataclass

@dataclass
class EscalationPolicy:
    labor_cost: float   # 人工评审的成本
    error_cost: float   # 无人监督下做错一次的成本

    @property
    def threshold(self) -> float:
        # tau* = 1 - c_l / c_w
        return 1.0 - self.labor_cost / self.error_cost

    def should_escalate(self, p_hat: float) -> bool:
        return p_hat < self.threshold

# 退款审批：一次错误的自动批准，代价是人工评审的 10 倍。
# tau* = 0.90 —— 除非模型有九成把握，否则一律上报。
refund_policy = EscalationPolicy(labor_cost=1.0, error_cost=10.0)
assert refund_policy.should_escalate(0.85)
assert not refund_policy.should_escalate(0.95)
```

仔细读这个式子：人工评审相对错误越便宜，阈值就越低；错误相对评审越便宜，阈值就越高。高风险动作（资金交易、删除）要设高阈值，日常内部查询设低阈值就行。这条判定规则住在你的领域模型里，不在 agent 里。

这是简单的那一半。难的那一半是 agent 的 `p_hat`，也就是它对自己正确性的自评。同一篇论文在五个真实决策域（贷款审批、酒店取消、内容审核等）上测了八个生产模型的隐式升级阈值。结果不太好看。

同一个模型家族内部，隐式阈值能差出 38 个百分点。GPT-5-nano 的隐式 `p*` 是 91%；同门的 GPT-5-mini 是 53%。Qwen3.5-9B 停在 56%，而且在多数领域都过度自信，自估准确率比实际准确率高出最多 41 个点。GPT-5-mini 在总体上校准得不错（自估 80% 对实际 78%），但在各个单独领域上背着 -38 到 +27 个点的、彼此抵消的偏差。[^6] 同家族的两个模型，不做重新测试就换掉，它们愿意上报的程度能差出三分之一条概率轴还多。

实际的推论是：**换模型却不重新刻画升级画像，就是一次安全性倒退。** 上线前，把任务难度按已知准确率分档，测每个档位上模型的升级率，找出模型跨过 50% 上报线的隐式 `p*`，再跟你成本最优的 `tau*` 对比。两者对不上，你要么是有一个过度自信、漏报的 agent，要么是有一个信心不足、把评审人淹在误报里的 agent。

发现缺口之后，什么真的能把阈值挪动。只在系统提示里写成本框架，对非推理模型几乎没用（Qwen3.5-9B 在 4:1 成本比下，决策准确率从 62.0% 只挪到 63.9%）。只开扩展思考也没用（61.9%）。思考和成本框架两者合用能到 78.8%。对 GPT-5-mini，单是成本框架就到 75.8%，两者合用 87.1%。用显式计算 `cost_ratio * error_rate` 的思维链目标做监督微调，在训练集、测试集和留出领域上都能接近 100%。[^6] 对多数团队来说微调是大炮打蚊子；思考加显式成本框架是默认选项，能拿到大部分收益。

Anthropic 2026 年 3 月上线的 Claude Code auto mode，是公开了数字的置信度门控升级里最大规模的生产部署。它分三档。[^7]

| 档位 | 包含什么 | 门禁 |
|---|---|---|
| 1 | 读取、grep、代码导航、用户配置的允许项 | 一律放行 |
| 2 | 项目内的文件写入（受版本控制覆盖） | 不经分类器直接放行 |
| 3 | shell、web fetch、外部工具、项目外的文件系统 | 两级分类器 |

第 3 档才是置信度升级真正跑起来的地方。第一级是一个快速的单 token 过滤器，倾向于拦截（对良性动作的误报率 8.5%）。第二级只对第一级标记出来的动作做思维链推理。整条管线在真实良性动作上误报率 0.4%，在真实“手太欠”的动作上漏报率 17%。[^7] 连续三次被拒、或一次会话里累计二十次被拒之后，模型会被停掉，并无条件呼叫人工；无人值守模式下直接终止进程。

17% 的漏报率是个诚实的数字。它比 `--dangerously-skip-permissions`（100% 漏报）好，比仔细的人工评审差。Anthropic 把两端都公开了，让你自己选站在哪。

## 交接：人到底看到了什么

一个评审人只拿到 `{"tool": "delete_record", "args": {"id": "4829f"}}`，别的什么都没有，他没法做真判断。他不知道用户要的是什么、agent 已经干了什么、删掉记录 `4829f` 能不能恢复。他会点批准，因为不批准的替代方案是卡住 agent 再去打扰用户。n8n 那个经典实现给的就是这个地板：`"The AI wants to use {{ $tool.name }} with the following parameters: {{ JSON.stringify($tool.parameters, null, 2) }}"`。[^8] 这不够。

一份生产级的交接载荷要带六样东西：

- **工具名和解析后的参数。** 被请求的字面动作。
- **用户的原始目标。** 评审人得知道当初要做什么，才能判断这个动作是不是在服务它。
- **本轮会话里 agent 已经做过什么。** 一行摘要列出之前的工具调用，让评审人看到轨迹。
- **针对这一次调用的单句理由。** agent 的规划轨迹，蒸馏成一句。它凭什么认为该用这个工具、这些参数？
- **可逆性。** 是幂等、可撤销，还是不可逆？要一个标记，不要一段话。
- **触发上报的风险信号。**“一封被标记为退款的邮件，置信度 0.62”比什么信号都不给有用得多。

![一张对比图：单薄的交接载荷只显示工具名和原始参数，完整载荷则额外给出用户目标、会话动作摘要、agent 的理由、可逆性标记，以及触发上报的风险信号](../../../ai-eng/part-7-agent-engineering/images/07-handoff-payload-anatomy.png)
*单薄的交接是框架默认给你的东西；完整的交接才是评审人做真判断真正需要的东西。*

Anthropic auto mode 分类器里有个反直觉的设计值得一说。分类器在读对话记录之前，会刻意剥掉 assistant 的叙述文字和工具结果，它只看到用户消息和光秃秃的工具调用命令。[^7] 这是有意为之的抗注入手段：agent 没法靠给自己的动作写一段有说服力的辩护，把分类器忽悠到错误决策上。代价是分类器没法评估来源。如果 agent 选的靶子是从上下文推断出来的、而不是用户明确点名的，分类器察觉不到这个偏差。

“用户说了句看起来像同意的话”和“用户真的授权了这个爆炸半径”之间的这段缝，就是大多数分类器漏报的来源。“清理一下这个 PR”对某些动作算同意，对强推 main 不算。分类器看到的是形似批准的证据，不去核实它是否覆盖了实际尝试的动作，于是放行。Anthropic 把这一点列为那 17% 漏报的首要来源，并说明到 2026 年 3 月为止没有干净的提示工程解法。[^7] 自动化分类补不上这段缝时，答案就是保守的信任边界（默认限定在当前 git 仓库，而不是用户的整个组织）、高风险动作要求显式授权，以及用环境层隔离兜底。HITL 这道门压在这些措施之上，不是拿来替代它们。

人工发起的评审的反面是 agent 发起的评审，而这恰恰是 HITL 里最没人关注的部分。同一份 Anthropic 测量发现，在最复杂的 Claude Code 任务上，agent 停下来请求澄清的次数，是被人打断次数的两倍多。主要原因：35% 是在几个候选方案之间给用户做选择，21% 是在收集诊断信息，13% 是在澄清含糊的请求。[^2] 一个能识别自身不确定性并把问题摆出来的 agent，是在提前替评审人干活。

Claude Agent SDK 通过 `AskUserQuestion` 工具暴露这件事：模型发出 1-4 个问题，每个问题 2-4 个结构化选项，你的应用把它们渲染出来，再通过处理其他任何审批时用的同一个 `canUseTool` 回调把答案送回去。[^5] 结构和普通门完全一样，只是由 agent 发起，而不是由规则触发。训练你的 agent 去用它。一个知道什么时候该停下来问的模型，比一个完美的外部分类器更有价值，因为它随任务复杂度一起扩展，而不是跟任务复杂度对着干。

## 参考文献

[^1]: Max McGuinness et al., Anthropic Engineering, "How we contain Claude across products", May 25, 2026. https://www.anthropic.com/engineering/how-we-contain-claude

[^2]: Miles McCain et al., Anthropic, "Measuring AI agent autonomy in practice", February 18, 2026. https://www.anthropic.com/research/measuring-agent-autonomy

[^3]: OpenAI, "Human-in-the-loop -- OpenAI Agents Python SDK", June 2026. https://openai.github.io/openai-agents-python/human_in_the_loop/

[^4]: LangChain, "Interrupts -- LangGraph Python docs", June 2026. https://docs.langchain.com/oss/python/langgraph/human-in-the-loop

[^5]: Anthropic, "Handle approvals and user input -- Claude Agent SDK", June 2026. https://console.anthropic.com/docs/en/agent-sdk/user-input

[^6]: Matthew DosSantos DiSorbo and Harang Ju, "Act or Escalate? Evaluating Escalation Behavior in Automation with Language Models", arXiv:2604.08588, March 31, 2026. https://arxiv.org/abs/2604.08588

[^7]: John Hughes et al., Anthropic Engineering, "How we built Claude Code auto mode: a safer way to skip permissions", March 25, 2026. https://anthropic.com/engineering/claude-code-auto-mode

[^8]: n8n, "Human-in-the-loop for AI tool calls", n8n docs, 2026. https://docs.n8n.io/advanced-ai/human-in-the-loop-tools/
