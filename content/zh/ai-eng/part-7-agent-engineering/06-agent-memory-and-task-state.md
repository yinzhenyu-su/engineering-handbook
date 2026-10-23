---
title: "智能体记忆与任务状态"
description: "智能体可以存放信息的四个地方，为什么大多数团队用错了地方，以及怎样设计出能在任务中途扛住 Pod 重启的状态。"
slug: 06-agent-memory-and-task-state
part: 7
chapter: 6
difficulty: intermediate
prerequisites:
  - 02-the-agent-loop
  - 03-tool-design
  - part-5-context-engineering/03-memory-and-state
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 智能体记忆与任务状态

你的编码智能体已经跑了九十分钟。它读了十二个文件，跑了三遍测试，重构迁移脚本的活儿干到一半。然后这个 Pod 被 OOM 杀掉了。编排器重启智能体，后者拿到一个空的状态对象和一个全新的上下文窗口，信心十足地从第一步重新开始，顺手把那份它压根不记得自己做过的半成品覆盖掉。

这就是默认行为。LLM 在设计上就是无状态（stateless）的：每次推理调用拿到一个上下文窗口、吐出 token，然后忘掉一切。一个要跑不止一步的智能体，必须在外面额外挂一层显式记忆，而这层记忆至少得同时承担三件不同的活儿。它需要单步推理内的工作记忆，需要多步任务的情节状态（episodic state），还需要把能活过整个任务的事实持久存下来。把这三件事揉进同一个机制，你造出来的系统会既慢得没法实时用，又浅得熬不过一次重启。

本章的任务就是在这几层之间划清界线，给你一条按数据逐项选层的规则，并说明当检查点（checkpointing）不再是可选项时，你的代码会发生什么变化。

## 智能体可以存放信息的四个地方

直觉上我们会把智能体记忆当成一件事。生产系统把它拆成四个不同的存储，各有各的延迟特征、变更规则和故障模式。

![四条水平分层纵向堆叠，最上面是薄薄的草稿板层，最下面是深不见底的长期存储层，左侧标注示例数据，右侧是延迟与持久性两条轴。](../../../ai-eng/part-7-agent-engineering/images/06-four-memory-tiers.png)
*顶部快而短暂，底部慢而持久。大多数智能体的 bug 都来自把数据放错了层。*

**草稿板（scratchpad）** 是单步推理内部的工作记忆。思维链 token、模型关于“下一步该调哪个工具”的计划笔记、上一次工具返回格式化后的观察结果。它活在上下文窗口里，调用一返回就蒸发，成本是与体积成正比的 token。

**核心记忆块（core memory blocks）** 是系统提示里常驻的小块命名区段：智能体人设、用户已知偏好、当前任务目标。Letta（MemGPT 的生产版后继）默认给每个块设 5,000 字符上限，并内置工具让智能体在运行期追加或替换块内容。[^1] 块永远可见，所以检索成本为零，但每次调用都要吃掉上下文窗口的预算。

**检查点状态（checkpointed state）** 是结构化的任务记录：哪些步骤完成了、它们产出了什么工具输出、智能体当前处在哪个阶段。它存在上下文窗口之外，放在以 `thread_id` 为键的数据库里，每个节点执行完自动写入，智能体从崩溃中恢复时重新读出来。正是这一层解决了本章开头那个编码智能体被 OOM 杀掉的问题。[^2]

**长期存储（long-term store）** 是跨会话、跨线程的那一层：关于用户的、应当跨越不相关任务保留下来的事实，组织知识，过往工作的情节摘要。底层技术和 RAG 一样（向量库、键值存储，有时是知识图谱），但这里的独特之处在 *写* 路径：由智能体决定提交什么。

大多数团队掉进的坑，是把上下文窗口当成唯一的层、把长期存储当成唯一的逃生通道。他们漏掉了中间那两层。一次九十分钟、二十次工具调用的智能体运行，如果不做检查点，就会尽职尽责地把完整消息历史塞进每一个 prompt，撑爆预算，然后在重启时照样全丢。

## 检查点器与存储：那条被模糊掉的界线

如果本章你只带走一条工程规则，就带走这条。检查点器（checkpointer）管的是 *本次任务运行中* 正在发生的事。存储（store）管的是 *关于这个用户* 为真的事。把两者搞混是智能体系统里最常见的架构错误，而搞错的代价只在出事的时候才显现。

概念上的判据是：如果明天为同一个用户全新起一次任务运行，我还要这份数据吗？要，那它就是存储数据：用户偏好、组织的编码规范、过往的情节摘要。如果只和当前这次运行有关（当前步骤号、部分工具输出、重试计数），那它就是检查点数据。

LangGraph 在 API 里把这一区分做得很明确。检查点器（`PostgresSaver`、`SqliteSaver`、`InMemorySaver`）在每个节点之后写入图状态的快照，以 `thread_id` 为键。[^2] 存储（`BaseStore`，带 `put`、`get`、`search`、`delete`）是另一个独立对象，按用户或组织划分命名空间，在所有线程之间持久存在。同一个智能体，两套完全不同的作用域规则。

```python
from typing import TypedDict

class AgentState(TypedDict):
    messages: list[dict]
    # 进检查点：线程作用域，任务内扛住崩溃
    task_id: str
    current_phase: str          # "planning" | "executing" | "verifying"
    completed_steps: list[str]
    tool_outputs: dict[str, str]
    # 任务开始时从存储加载，不逐步进检查点
    user_preferences: dict[str, str]

def planning_node(state: AgentState) -> dict:
    """Returns a partial update; the framework merges it into the checkpoint."""
    return {
        "current_phase": "executing",
        "completed_steps": state["completed_steps"] + ["planning_done"],
    }
```

微妙的那一行是 `user_preferences`。它 *在* 状态 schema 里，是因为智能体推理时要读它，但它不是任务产出的。上游一个 `load_memory_node` 在任务开始时查存储把它种进去；智能体执行期的工具调用不会直接写回它。把这两层混在一起，你就会遇到用户偏好被悄悄回滚到三十天前的快照，因为检查点里根本没包含它们。

## 哪些该写下来，哪些该留在上下文里

对每一份数据，你有四个选择。只要问对那四个问题，决策规则就是机械的。

| 层级 | 何时放在这里 | 例子 |
|---|---|---|
| 上下文窗口（常驻） | 小（<2,000 字符）、极少变更、每步都要 | 智能体人设、当前任务目标 |
| 草稿板 | 只属于当前这一步的临时推理 | 思维链、工具调用计划笔记 |
| 检查点状态 | 任务内可变的进度，必须扛住崩溃，线程作用域 | 当前阶段、已完成步骤、工具输出 |
| 长期存储 | 跨会话的事实，体积大或持续增长，靠查询检索 | 用户历史、组织知识、过往情节 |

Letta 上下文层级文档里的规则很直接：小数据用记忆块留在上下文窗口里；只有数据很大或者跨会话边界时，才升级到外部存储。[^1] LangGraph 记忆指南给出的推论就是上一节那个判据：线程作用域的数据放检查点器，跨线程的数据放存储。[^3]

具体一点：

```python
from enum import Enum, auto

class StorageTier(Enum):
    CONTEXT_WINDOW = auto()
    CHECKPOINTED_STATE = auto()
    LONG_TERM_STORE = auto()

def decide_storage_tier(
    data_size_chars: int,
    crosses_sessions: bool,
    mutation_rate: str,
    needed_every_step: bool,
) -> StorageTier:
    if data_size_chars > 4_000 or crosses_sessions:
        return StorageTier.LONG_TERM_STORE
    if needed_every_step and mutation_rate == "high":
        return StorageTier.CHECKPOINTED_STATE
    if needed_every_step and data_size_chars < 2_000:
        return StorageTier.CONTEXT_WINDOW
    return StorageTier.CHECKPOINTED_STATE
```

把任一候选数据丢进这四个问题里过一遍。把 50KB 的文档放进上下文窗口，代价是一回事；把快速变化的步骤计数器放进长期存储更糟，因为你在透过错的抽象写数据，你的检索查询会永远被陈旧的运行状态污染。

二阶规则是关于预算的。Mem0 论文显示，在 26,000 token 对话的 LOCOMO 基准上，全上下文处理的 p95 是 17.1 秒；而一条把检索上下文压在 7,000 token 以内的抽取式记忆流水线只需 1.44 秒，延迟降低 91%，准确率只小幅下降（LLM-as-a-Judge：67 对 73）。[^4] 在长时间运行智能体的量级上，这个比例就是交互式产品和批处理作业的差别。

## 检查点：熬过 Pod 重启的那个技巧

回到那个被 OOM 杀掉的编码智能体。修法只有一行配置，而大多数教程都跳过了它。

LangGraph 的默认行为是不带检查点器。如果你调 `graph.compile()` 时不传，整个状态就活在进程内存里，随进程一起死。这在开发阶段看不出来，因为单进程测试从不在任务中途重启。放到生产就是灾难，那里 Pod 会被驱逐、被 OOM 杀掉，或者在发版时滚动替换。社区里那句话总结得到位：“LangGraph 把内存中的情况做得太顺滑，以至于你忘了还有记忆这回事，然后生产环境会提醒你。”[^5]

持久化检查点器改变的是故障模式。图中每个节点执行完之后，框架会把完整状态字典、各通道版本号和一个单调递增的检查点 ID 序列化进配置好的后端。[^2] 崩溃时你调 `graph.invoke(None, config={"configurable": {"thread_id": same_id}})`，执行就从最后完成的那个步骤接着走，而不是从头来。代码路径不变；框架读这个 `thread_id` 的最新检查点，继续跑。

生产级做法是用 `PostgresSaver`（单进程的话用 `SqliteSaver`），在应用启动路径里初始化一次：

```python
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg_pool import ConnectionPool

pool = ConnectionPool(conninfo="postgresql://...", max_size=10)
checkpointer = PostgresSaver(pool)
checkpointer.setup()  # 首次运行时建表；幂等

graph = builder.compile(checkpointer=checkpointer)

# 现在每次调用都会按 thread_id 持久化状态
config = {"configurable": {"thread_id": "task-001"}}
result = graph.invoke({"messages": [...], "task_id": "task-001", ...}, config)
```

检查点这套 API 还顺带打开了人机协同（human-in-the-loop）模式，[人机协同](../../../ai-eng/part-7-agent-engineering/07-human-in-the-loop.md) 里会专门讲：用 `interrupt_before=["risky_node"]` 编译，执行就停在那里，检查点写入，审阅这次运行的人可以调 `graph.update_state(...)` 修正中间值再恢复。智能体逻辑里不需要任何协调代码。同一套机制同时覆盖崩溃恢复、时间旅行式调试和审批关卡。

天真地打开检查点的团队会踩两个坑。第一个是检查点膨胀：如果你的状态 schema 里装着大对象（整篇文档正文、base64 图片、很长的工具输出），每次写检查点都要把这些字节序列化进 Postgres。一个十步的任务，每次检查点写 200ms，就白添两秒纯 I/O。修法是状态里只放引用（S3 key、文档 ID），只在真正需要它的节点里取实体。第二个是生产环境的检查点器选型：`InMemorySaver` 是给测试用的；任何可能重启的部署，你都需要一个真正的后端。[^2]

## 扁平列表还是任务树？

多数任务是线性的：规划、一串步骤、然后验证。检查点状态里一个扁平的 `completed_steps: list[str]` 字段就够了。Mem0 那套摘要加一个步骤计数器足矣。

任务一旦分叉，它就不够了。一个编码智能体跑测试、发现失败、回头改早先某一步、再从那里重跑，它的历史没法用一个扁平列表表示，否则它自己都分不清哪个步骤的哪个版本才是当前的。Task Memory Engine 论文用任务记忆树（Task Memory Tree）把这件事形式化：每个节点存 `{action, input, output, status, parent_id, children, dependencies}`，构建上下文时提示合成器只遍历当前活跃的那条根到叶路径。[^6] 在一个带纠错的六轮表单填写任务上，相比朴素地拼接全量历史，这省了约 19% 的 token，因为模型再也不会重复看到被放弃的分支。

升级规则很简单。流程可预测、线性的任务用扁平列表。出现条件分叉、会让先前工作作废的重试路径，或者那种要覆盖先前步骤但不能抹掉审计痕迹的人工纠错，就换树。别第一天就上 DAG 库；十行递归 Python 遍历一个 `dict` 装节点，在它撑不住之前一直够用。

## 与用户记忆的边界

[记忆与状态](../part-5-context-engineering/03-memory-and-state.md) 详细讲了面向用户的记忆：滑动窗口、摘要缓冲、带 ADD/UPDATE/DELETE/NOOP 分类的事实抽取、GDPR 删除，以及防止长期存储漂移成一堆噪声的淘汰策略。这里不要重新实现一遍。两章之间的界线很清楚：第 5 部分讲的是 *产品跨会话记住了用户的什么*；本章讲的是 *智能体记住了它此刻正在做的这个任务的什么*。

两者之间的交接点是加载点和保存点。任务开始时，你去长期存储（第 5 部分讲的）里查相关用户事实，种进智能体状态。任务结束时，智能体学到的、值得留下的东西（这个用户偏好 Python 而不是 Ruby，这个组织的部署脚本需要 `--no-cache`）走一遍事实抽取流水线（也是第 5 部分），再写回去。任务过程中，唯一的记忆工作就是上面那四层。

智能体这一侧还有两个特有的故障模式值得点出来。

**工具返回带来的记忆污染。** 当工具返回攻击者可控的文本，那段文本会进入上下文，被你的推理步骤处理，如果你接了自动事实抽取，还可能被写进长期存储。下一次会话，它就作为事实被检索出来。MEMTIER 论文在扁平文件式记忆系统上测到，运行 72 小时后工具执行成功率下降 14 个百分点，原因与此相关。[^7] 修法在 [工具设计](./03-tool-design.md) 里：把每一份工具输出都当不可信输入，抽取前先清洗，并在每条记忆记录上存来源（`source_tool`、`source_call_id`），这样被污染的条目日后可以清掉。

**共享块的并发写。** 多个子智能体往同一个 Letta 式记忆块里写时，后写的赢，先写的无声消失。[^1] 避免这个的模式是一块一写者：给每个子智能体自己的草稿板块，然后在主管层合并结果。对于权威的共享事实，把块标成只读。

验证架构是否正确的办法，最可靠的是故意在任务中途杀掉进程。如果智能体从停下的地方接着跑，用户偏好完好，进行中的工作正确，那说明四层各司其职。如果少了什么，那份数据就是放错了层。

## 参考文献

[^1]: Letta，"Memory blocks (core memory)"，Letta 文档，访问于 2026 年 6 月。https://docs.letta.com/guides/core-concepts/memory/memory-blocks

[^2]: LangChain，"Checkpointing: state persistence, crash recovery, and time-travel debugging in LangGraph"，LangGraph 文档，2026 年 3 月。https://langchain-ai-langgraph-40.mintlify.app/concepts/checkpointing

[^3]: LangChain，"Memory: short-term and long-term memory in LangGraph agents"，LangGraph 文档，2026 年 3 月。https://langchain-ai-langgraph-40.mintlify.app/guides/memory

[^4]: Chhikara, Khant, Aryan, Singh, Yadav，"Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory"，arXiv:2504.19413，2025 年 4 月。https://arxiv.org/abs/2504.19413

[^5]: learnwithparam.com，"LangGraph persistence and thread models in production"，2025 年 2 月。https://www.learnwithparam.com/blog/langgraph-persistence-thread-models-production

[^6]: Ye，"Task Memory Engine: Enhancing State Awareness for Multi-Step LLM Agent Tasks"，arXiv:2504.08525，2025 年 4 月。https://arxiv.org/abs/2504.08525

[^7]: Sidik and Rokach，"MEMTIER: Tiered Memory Architecture and Retrieval Bottleneck Analysis for Long-Running Autonomous AI Agents"，arXiv:2605.03675，2026 年 5 月。https://arxiv.org/abs/2605.03675
