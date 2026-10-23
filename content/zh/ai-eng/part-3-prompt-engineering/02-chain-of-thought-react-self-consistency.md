---
title: "思维链、ReAct 与自一致性"
description: "从 2023 年的提示词文献里活下来的三项技术，以及推理模型带来的规则反转：各自何时该用，何时该删掉。"
slug: 02-chain-of-thought-react-self-consistency
part: 3
chapter: 2
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/04-reasoning-models
  - 01-zero-shot-and-few-shot
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 思维链、ReAct 与自一致性

三年来被贴进每一场 AI 演讲的那句建议——“直接让它一步一步思考”——在你团队正在调用的模型里，越来越大的比例上已经是错的。在 Gemini Flash 2.5 上显式做思维链提示，博士级问题的准确率掉了 3.3 个百分点，响应时间比不带这句后缀的同一提示拉长了 20% 到 80%。[^1] 同一个模型。同一批问题。答案更差，更慢，还花得更多。

但这并不是全部。换成 Claude Sonnet 4.6 或 GPT-4o 这类非推理模型跑同样的算术应用题，“let's think step by step”仍然像 2022 年的论文说的那样有效。技术本身没有坏。是模型分成了两类，其中一类不再需要它。

本章讲的是熬过 2023 到 2024 那一轮研究的三种提示词技术，每一种对模型做了什么，以及决定哪类模型配哪种技术的那条规则。规则对了，你就能在收益真实存在的地方保住收益；规则错了，你白交一笔延迟税。

## 两类模型，以及同一句提示为什么效果相反

标准的指令微调模型，比如 GPT-4.1、Claude Sonnet 4.6 或 Gemini 2.0 Flash，没有内置的草稿纸。你给它一道多步算术题，它唯一的工作记忆就是自己已经写出来的 token。直接给答案意味着一次前向传播，中间结果没有地方推导。思维链（chain-of-thought，CoT）提示则是让模型先把步骤写出来，等于给它买了一张草稿纸。[^2]

推理模型（reasoning model）反过来：OpenAI 的 o 系列、开了自适应思考的 Claude，或者 DeepSeek-R1。它用强化学习训练过，会在可见答案之前的隐藏 token 里跑自己的思维链。那些 token 会探索不同路径、回退、验证。等你看到回复时，推理已经做完了。[^3] 完整机制见[推理模型](../part-1-how-models-behave/04-reasoning-models.md)；这里关心的是它对提示词意味着什么。

你对推理模型说“think step by step”，等于对一个已经在思考中的模型说：你要按这种方式想。这条指令会进入模型的内部规划过程，把它框住。少样本示例（few-shot exemplar）的干扰更硬：它把模型锚定在你示例所暗示的某种推理形态上，而那种形态往往比模型自己会探索出来的更差。[^4][^5]

![左右对比：左边是标准模型配上思维链提示，产生一条很长的可见推理链，最后给出答案；右边是推理模型，同一句提示与隐藏的内部推理过程相撞，产出一个更差、更慢的答案。](../../../ai-eng/part-3-prompt-engineering/images/02-cot-inversion.png)
*同一句提示帮了一类模型，害了另一类。真正承重的决策是模型类别，不是提示词。*

识别这一步才是元规则。选提示词技术之前，先说出模型属于哪一类。模型卡里出现“o-series”“extended thinking”“adaptive thinking”“DeepSeek-R1”或“Gemini 2.5 thinking”，就是推理模型；其余的算标准模型。[^4]

## 思维链：草稿纸的把戏

Wei 等人在 NeurIPS 2022 发表了最初的思维链（CoT）论文。想法很小，结果很大：给模型看几个带推理步骤的完整样例，它就会模仿这个格式，在作答前写出自己的步骤。在 GSM8K 算术应用题上，540B 的 PaLM 模型用 8 个示例的 CoT 提示，赢过了带验证器的微调 GPT-3。[^2] 一年后，Kojima 等人证明连示例都不需要。在提示词后面接一句“Let's think step by step”，一个示例都不给，也能拿到大部分收益。这就是零样本思维链（zero-shot CoT）。[^6]

一个典型的少样本提示长这样：

```python
import anthropic

client = anthropic.Anthropic()

examples = [
    {
        "question": "Roger has 5 tennis balls. He buys 2 cans of 3 balls each. How many does he have?",
        "cot": "Roger starts with 5 balls. Each can has 3 balls; 2 cans is 2 * 3 = 6. Total: 5 + 6 = 11.",
        "answer": "11",
    },
]

messages = []
for ex in examples:
    messages.append({"role": "user", "content": ex["question"]})
    messages.append({"role": "assistant", "content": f"{ex['cot']}\n\nAnswer: {ex['answer']}"})
messages.append({"role": "user", "content": "Janet has 15 apples. She gives away 7. How many remain?"})

# 这里必须用标准模型。不要把这套模式粘到 claude-opus-4-8 或 o3 上。
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=messages,
)
```

关于 CoT 到底在哪儿有用，有两点要说。第一是模型规模：原论文发现，CoT 大约要在 1000 亿参数以上才稳定见效；更小的模型写出的推理链并不能改善答案。[^2] 第二是任务类型。Sprague 等人 2024 年的元分析覆盖了 20 个数据集、14 个模型上的 100 多篇 CoT 论文，结论毫不含糊：涉及数学或符号推理的任务，CoT 收益显著；其他任务几乎为零。在 MMLU 上，只要问题或回答里不含等号，直接生成就和 CoT 打平——这是元分析里一句直白的总结：CoT 起作用，主要是因为让模型去执行符号步骤。[^7]

所以标准模型的规则很窄。多步数学、逻辑或符号问题，用零样本 CoT。推理链不稳定或格式跑偏时，升级到少样本 CoT。事实检索、分类、情感分析，别用。CoT 会多出 token，也就是多出延迟和账单；按 Wharton 2025 年的实测，非推理模型加 CoT 比同一提示不加要慢 35% 到 600%。[^1] 一个要花你 5 到 15 秒的“免费”技巧，并不免费。

推理模型的规则更短：别用。截至 2026 年 6 月，OpenAI 的官方指引原文是：“Avoid chain-of-thought prompts. Since these models perform reasoning internally, prompting them to 'think step by step' or 'explain your reasoning' is unnecessary.”[^4] Anthropic 的提示词指南用另一种说法讲了同一件事：“A prompt like 'think thoroughly' often produces better reasoning than a hand-written step-by-step plan.”[^5] Wharton 的实测数据支持这个策略：显式 CoT 提示让 o3-mini 涨 2.9%、o4-mini 涨 3.1%，代价是 20% 到 80% 的延迟；Gemini Flash 2.5 反而掉了 3.3%。[^1] 这些没有一个是赚的。

推理模型表现不及格时，解法不是加更多提示词脚手架，而是把问题描述收紧、把成功标准写明确、把 `reasoning.effort`（OpenAI）或 `effort`（Anthropic）参数调高。[推理模型](../part-1-how-models-behave/04-reasoning-models.md)那一章把这个力度旋钮讲透了。

## 自一致性：投票，别信单条链

CoT 有一个脆弱的失效模式。如果第一步算术就错了，后面每一步都建立在错误的数字上，模型会自信地一路走到错误答案上。贪心解码不会回头。

Wang 等人在 ICLR 2023 用一个便宜到极点的提准办法解决了它。同一句 CoT 提示，在温度大于零的条件下跑几次，从每次结果里抽出最终答案，然后多数投票。[^8] 不同采样链在不同步骤上犯不同的错：错误答案散开，正确答案聚拢。

```python
from collections import Counter
import re

def extract_final_answer(text: str) -> str:
    """Pull the last number out of a CoT response."""
    matches = re.findall(r"\b\d+(?:\.\d+)?\b", text)
    return matches[-1] if matches else ""

def majority_vote(responses: list[str]) -> str:
    answers = [extract_final_answer(r) for r in responses]
    counter = Counter(a for a in answers if a)
    return counter.most_common(1)[0][0] if counter else ""

# 用法：以约 0.7 的温度调用模型 k 次，收集所有回复，
# 再跑 majority_vote(responses)。最终答案就是那个共识。
```

在标准模型上，收益是真实的，而且不小。在 PaLM 540B 上，CoT 加自一致性（self-consistency）把 GSM8K 准确率提高 17.9 个百分点、SVAMP 提高 11.0、AQuA 提高 12.2。[^8] 这些是 2023 年在 2022 年的模型上测出的基准数字，但这套方法在不同模型家族和规模上都能复现。

代价与采样次数成正比。原论文用的是 k=40，也就是一个答案要调 40 次，成本 40 倍。生产团队应该从 k=5 到 10 起步，在自己的评测集上测，只有在准确率收益能覆盖成本时才往上加。大部分收益在远未到 40 的时候就饱和了；后续关于提前停止（early stopping）的工作——一旦出现明显多数就停止采样——能用几分之一的成本拿回大部分收益。[^9]

自一致性有硬性限制。它需要一个可抽取的单一最终答案：一个数字、一个标签、一个结构化字段。开放式生成它一点用没有，两篇文章之间没法“投票”。模型以一致的方式出错时它也失效：多数派可能收敛到贪心链本来也会选的同一个错误答案。而在推理模型上，边际准确率收益小到几乎不值得为 k 次采样付费——那个模型本来就在内部探索多条路径。推理模型上更干净的做法是把它的 effort 等级调高，而不是把它调五次。[^1]

## ReAct：变成智能体循环的那个模式

第三种技术解决的是另一个问题。CoT 在模型已经知道事实时才有效。不知道的时候，CoT 链会编：模型填上听起来合理的数字和日期，剩下的推理是严谨的胡说。

Yao 等人 ICLR 2023 的论文提出把推理步骤和工具调用交替穿插。模型轮流输出几种块：`Thought` 说明下一步做什么，`Action` 调用一个工具（比如维基百科搜索或计算器），`Observation` 装工具返回的结果。每条观察把下一个想法落到真实事实上。模型输出 `Finish[answer]` 时循环结束。[^10]

在原始基准上，ReAct 同时赢过纯 CoT 和纯动作预测的基线。在基于文本的家务任务基准 ALFWorld 上，ReAct 的成功率比最强的强化学习基线高 34 个百分点。在商品搜索任务 WebShop 上，差距是 10 个百分点，而上下文里只给了一到两个示例。[^10]

这是历史。下面是它留在生产环境里的部分。

纯文本的 `Thought / Action / Observation` 格式基本是遗迹。它存在是因为 2022 年的模型没有结构化的工具调用 API，想让模型调用工具，唯一的办法是用少样本示例教它这套约定。现在的模型有原生函数调用，[函数与工具调用](../part-2-working-with-model-apis/03-function-and-tool-calling.md)那一篇讲过，供应商的 API 自己负责这种交替。你把工具定义成 JSON schema，模型返回结构化的工具调用对象，你的代码执行它，再把结果作为工具消息喂回去。没有 `Thought:` 前缀要格式化，也没有解析器要照看。对推理模型来说，内部思维链已经扮演了可见 `Thought` 块的角色，一个朴素的函数调用循环就能追平甚至超过原始 ReAct 轨迹。[^4][^10]

活下来的是架构，不是语法：把推理和有依据的动作交替起来，别让模型假装它知道必须去查的事实。你会碰到的每个智能体框架（LangGraph 的 `create_react_agent`、OpenAI 的 Agents SDK、Anthropic 的工具使用循环）都是这个想法的后代，只是打包成图，而不是提示词。

有一个失效模式从原论文一路带到现代智能体：动作循环。如果同一个查询工具返回同样的结果，而模型不更新计划，它就会拿相同的参数再调一次、再调一次。修法是机械的，不是玄学。给循环设边界。LangGraph 暴露了 `recursion_limit`；OpenAI 允许你限制轮数；不管你用哪个框架，都设一个硬上限（10 到 15 步是合理的起点），并把重复的相同调用设为停止条件。一个会放弃的有界循环，比一个在有人发现之前把账单跑到四位数的无界循环便宜。

## 哪类模型配哪种技术

三种技术，两类模型，一张表：

| 技术 | 标准模型 | 推理模型 |
|---|---|---|
| **零样本 CoT**（“think step by step”） | 数学、逻辑、符号任务的默认选项。检索和分类跳过。 | 不要加。按 OpenAI 和 Anthropic 自己的指引，它会干扰内部推理。[^4][^5] |
| **少样本 CoT** | 零样本链不稳定时使用。注意示例与真实查询之间的分布不匹配。 | 不要加。它会把模型锚定在比它自己会探索的更差的分解上。[^4][^11] |
| **自一致性** | 封闭式答案的准确率比成本更重要时，用 k=5 到 10。远在 k=40 之前就饱和。 | 改为调高 `reasoning.effort`。模型本来就在内部采样。 |
| **ReAct（文本格式）** | 只在没有原生函数调用的小模型或旧模型上用。 | 不要用。用函数调用 API。 |
| **ReAct（架构）** | 交替模式以智能体循环的形式延续下来。只要模型需要它自己没有的、有依据的事实时就用。 | 同样。内部链取代了可见的 `Thought` 块。 |

如果这一章只让团队养成一个习惯，那就是识别这一步。伸手去拿这些技术之前，先把模型类别说出来。这一步定了，规则就干脆。2026 年最贵的提示词错误，就是把所有模型一视同仁，然后纳闷为什么同一句提示在 Sonnet 上给你赢了 18 个点，在 Flash 2.5 上却让你掉了 3 个。

## 参考文献

[^1]: Meincke, L., Mollick, E.R., Mollick, L., Shapiro, D., "The Decreasing Value of Chain of Thought in Prompting," Wharton Generative AI Labs Technical Report, June 8, 2025. https://gail.wharton.upenn.edu/research-and-insights/tech-report-chain-of-thought/

[^2]: Wei, J. et al., "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models," NeurIPS 2022. https://arxiv.org/abs/2201.11903

[^3]: DeepSeek-AI, Guo, D. et al., "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning," Nature vol. 645, 633-638 (2025). https://arxiv.org/abs/2501.12948

[^4]: OpenAI, "Reasoning best practices," OpenAI Platform Documentation, as of June 2026. https://platform.openai.com/docs/guides/reasoning-best-practices

[^5]: Anthropic, "Prompting best practices," Anthropic Documentation, as of June 2026. https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices

[^6]: Kojima, T., Gu, S.S., Reid, M., Matsuo, Y., Iwasawa, Y., "Large Language Models are Zero-Shot Reasoners," NeurIPS 2022. https://arxiv.org/abs/2205.11916

[^7]: Sprague, Z. et al., "To CoT or not to CoT? Chain-of-thought helps mainly on math and symbolic reasoning," ICLR 2025. https://arxiv.org/abs/2409.12183

[^8]: Wang, X. et al., "Self-Consistency Improves Chain of Thought Reasoning in Language Models," ICLR 2023. https://arxiv.org/abs/2203.11171

[^9]: "Early-stopping Self-Consistency for Multi-step Reasoning," ICLR 2024. https://iclr.cc/virtual/2024/poster/17848

[^10]: Yao, S. et al., "ReAct: Synergizing Reasoning and Acting in Language Models," ICLR 2023. https://arxiv.org/abs/2210.03629

[^11]: Nori, H. et al., "Exploration of Run-Time Strategies for Medical Challenge Problems and Beyond," arXiv:2411.03590, November 2024. https://arxiv.org/abs/2411.03590
