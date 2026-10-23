---
title: "模板与链式调用"
description: "把提示词当成有类型的代码来写，以及你每周都要做的那次取舍：一个大提示词，还是几个小提示词串成链。"
slug: 03-templates-and-chaining
part: 3
chapter: 3
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/00-calling-models
  - part-2-working-with-model-apis/02-structured-outputs
  - part-2-working-with-model-apis/07-prompt-caching
  - 00-anatomy-of-a-production-prompt
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 模板与链式调用

上个季度，一位同事改写了线上提示词里的一行。原本的 “Output strictly valid JSON” 变成了 “Always respond using clean, parseable JSON”。改动看着只是措辞。schema 没坏，任何测试也没报错。它只是开始吐出尾随逗号，并在边界输入上偶尔漏掉必填字段，结果把所有下游解析器悄悄弄坏，九天后才有人发现。[^1]

值班工程师的第一个问题很自然：*现在跑的是哪个版本的提示词，它和上周有什么不同？* 那段提示词是 `app.py` 里的一个字符串常量。没有 diff，没有 PR，没有可回滚的目标。修复只花了九十秒，找出要修什么却耗掉了大半个下午。

这起事故就是“把提示词当数据库查询来对待”的全部理由。硬编码的提示词字符串，等于用字符串拼接写 SQL：没有类型安全，没有 diff 历史，没法测试，没法回滚。修法并不复杂：把提示词写成一个带类型输入的函数，返回 API 请求体，然后让它和你要发布的其他东西一样，接受同一套工程纪律。

## 要版本化的是提示词资产，不是提示词字符串

第一步，是别再把它当成一段文本。你真正要做版本管理的是一个四件套：模板文本、模型标识、采样参数（`temperature`、`max_tokens`、`top_p`），以及期望输出的 schema。temperature 0 跑的和 temperature 0.9 跑的，哪怕字面一模一样，也是两个不同的产品。排查一次幻觉需要这四样都留下；你没记录下来的东西，就回滚不了。[^2]

最小可用的模板就是一个 Python 函数：

```python
from string import Template

SYSTEM = Template(
    "You are a $role. Respond in $language. Be concise."
)
USER = Template(
    "<document>$document</document>\n\n"
    "Summarize the document above in $max_sentences sentences."
)

MODEL_CONFIG = {
    "model": "claude-opus-4-8",
    "max_tokens": 512,
    "temperature": 0.0,
}

def build_summary_request(
    role: str, language: str, document: str, max_sentences: int
) -> dict:
    return {
        "system": SYSTEM.substitute(role=role, language=language),
        "messages": [{
            "role": "user",
            "content": USER.substitute(
                document=document, max_sentences=max_sentences
            ),
        }],
        **MODEL_CONFIG,
    }
```

相比字符串常量，这个写法给你三样东西。变量缺失会在调用时抛 `KeyError`，而不是默默拼出一段残缺的提示词。模型配置跟着模板一起走，改 temperature 会以代码 diff 的形式出现。函数还是可测的：你可以写单元测试，断言 `build_summary_request` 产出的字节和你预期完全一致，全程不用真的调模型。

LangChain 的 `ChatPromptTemplate` 和 Mirascope 的类型化提示词把同一个想法包上了更多结构（输入变量声明、Pydantic 类型化输入、多消息格式化），但它们只是同一个原语的展开。选适合你技术栈的那个就行。不可让步的一条是：提示词是函数，不是字符串字面量。这个四件套存到哪儿（源码仓库、数据库、专门的注册中心）、非工程师怎么迭代它，是[提示词版本管理](./04-prompt-versioning.md)要讲的内容；眼下记住一条规则就够了：它是代码。[^2]

## 一个大提示词，还是几个小提示词

提示词变成代码之后，下一个决策是结构性的。给你一个任务：拿一份长文档，抽取实体和日期，判定文档类型，再写一段两句话的摘要，点出其中的财务风险敞口。有两种交付方式。写一个大提示词，一次干完这四件事。或者写四个提示词，顺序执行，把每一步的输出喂给下一步。

单提示词版本在运维上更简单：一次 API 调用、一行日志、一个来回的延迟。链式版本代码更多、延迟更高、token 也更多（每一步都要单独付一次调用的固定开销）。所以链式必须用它换来的东西，证明这份复杂度值得——而且得是那坨单体给不了的东西。

它能给三样。**逐步骤的错误定位**：出问题时你知道是哪一步产出了坏输出，而不是对着四千 token 的一坨发呆。**独立重试**：重跑一次 200 token 的抽取，比重跑一个 3,000 token 的巨型提示词便宜。**校验闸门**，这条最关键：在两步之间插一道程序检查，断定输出格式合规，下一步才看得到它。

没有闸门，链式最坏的失效模式是静默的。步骤 1 返回一段畸形 JSON 字符串。步骤 2 收到它，把它当成事实，因为 LLM 不会因为输入有错就抛异常；它们会产出看起来合理、却把那批坏数据揉进去的输出。步骤 3 又收到步骤 2 那份自信的错误综合。2026 年一项关于多智能体流水线的研究把这种现象记录为系统性而非偶发：早期步骤的小错误会被下游引用和复用，最终在整条链上生成作者所称的“集体虚假共识”。[^3] 输出是错的，而且自洽——这是最糟糕的组合。

修法是在步骤之间放一个解析器：

```python
import json
from dataclasses import dataclass

@dataclass
class StepResult:
    step: str
    output: str
    ok: bool

def call_llm(system: str, user: str) -> str:
    # 真实实现：anthropic.Anthropic().messages.create(...)
    return '{"entities": ["Alice", "Bob"], "action": "signed contract"}'

def step_extract(document: str) -> StepResult:
    raw = call_llm(
        system="Extract entities and actions as JSON: "
               "{entities: [...], action: string}",
        user=document,
    )
    try:
        parsed = json.loads(raw)
        ok = isinstance(parsed.get("entities"), list) and "action" in parsed
    except json.JSONDecodeError:
        ok = False
    return StepResult(step="extract", output=raw, ok=ok)

def step_summarize(extracted_json: str) -> StepResult:
    raw = call_llm(
        system="Summarize the following JSON facts in 2 sentences.",
        user=extracted_json,
    )
    return StepResult(step="summarize", output=raw, ok=True)

def run_chain(document: str) -> list[StepResult]:
    r1 = step_extract(document)
    if not r1.ok:
        raise ValueError(f"Extract gate failed: {r1.output}")
    return [r1, step_summarize(r1.output)]
```

那句 `if not r1.ok` 就是全部要害。它拒绝把一段过不了 JSON 解析的字符串，交给正要把它当事实用的下一步。在生产里，你会在抛出之前先用改进过的提示词重试抽取步骤，但结构性的性质是：任何东西想跨过步骤边界，必须先被解析和校验。[^4]

[五种工作流模式](../../../ai-eng/part-7-agent-engineering/01-the-five-workflow-patterns.md)梳理了线性链之外的变体，包括路由、并行化和评估者-优化者循环。每种模式都是同一个原语（类型化步骤、闸门、下一步），拼成不同的形状。

## 链式调用真正占优的场景

尽管干净分层的说法很诱人，默认答案依然是：一个大提示词。单次调用构建更便宜、调试更便宜，延迟还只是零头。链式只在少数几种情况下赚回它的复杂度：

- 任务里有两个或更多真正不同的阶段，输出格式也不一样（先抽结构化事实，再写散文）。单提示词得在一次前向传播里切换推理模式；链式可以让每一步独占模型的全部注意力。
- 中间输出需要程序检查、人工审核或条件分支，下一步才敢跑。
- 在你自己的评测集上，单体的准确率实测低于链式。不是“我们觉得它应该低”，是数字上更低。

而在另外两种情况下它会输，有时输得很难看，而这两种情况的研究结论格外清晰。

第一种是推理原生模型。沃顿 GAIL 的《思维链价值递减》研究（2025 年 6 月）测量了在 o3-mini、o4-mini 和 Gemini Flash 2.5 之上再叠一层显式思维链提示的效果。准确率变化：+2.9%、+3.1%，以及*负* 3.3%。延迟开销：20% 到 80%。[^5] 这些模型内部本来就在逐步推理；在外面再焊一层链式，只增加了来回次数，却没有可改进的推理面。如果你用的是推理模型，直接把问题问它，让它自己想。

第二种是人为切分。任务本来没有分明阶段，你却因为链式看起来更整齐而硬拆，那么协调开销（序列化、传递、反序列化、重建上下文）会把收益吃干净。生产中的信号是：单步准确率不错，端到端准确率并不比单体好，延迟和成本却是 3 倍。[^4] 修法是合并步骤，直到每一步都在干真活。

| 默认选择 | 什么时候升级到链式 |
|---|---|
| 一个大提示词 | 任务有 2 个以上阶段，推理方式或输出形态不同 |
| 一个大提示词 | 中间输出需要校验、分支或人工审核 |
| 一个大提示词 | 你的评测集显示链式在端到端上胜过单体 |

链长上限是 3 到 5 步。再往上，级联风险和状态管理复杂度会以比收益更快的速度叠加。一条链因为每遇到一个新边界情况就长出一个新步骤而不断变长，那是在积累复杂度，不是在管理复杂度。[^4]

诚实的检验是那道没人做的对照：同一任务、同一评测集，单体对链式，端到端测量。单步准确率可以很高，端到端却*低于*单体，因为协调开销把收益吃掉了。大多数团队凭直觉选。选得准的团队靠测量。

## 缓存改写了成本算式

链式有一个单体没有的结构性质：每一步开头的系统提示词，与上一步逐位相同。如果你把这个前缀组织好，第一步之后的每一步都能从提示词缓存读取，价格大约是基础输入费率的 10%。哪一步没组织好，就付全价。[^6]

规则与[提示词缓存](../part-2-working-with-model-apis/07-prompt-caching.md)那一篇完全一样：稳定内容放前面，变动内容放最后。

```python
from string import Template

STABLE_SYSTEM = """You are a financial analysis assistant.
Rules:
1. Always cite the source document.
2. Round all figures to two decimal places.
3. Flag any figures older than 12 months.
"""
# 每一步、每个会话都逐位相同。可缓存。

USER_TEMPLATE = Template(
    '<document source="$source" date="$date">$content</document>\n\n$question'
)
# 所有变动都在这里，位于缓存前缀之后。

def build_request(source: str, date: str, content: str, question: str) -> dict:
    return {
        "model": "claude-opus-4-8",
        "max_tokens": 512,
        "system": STABLE_SYSTEM,
        "messages": [{
            "role": "user",
            "content": USER_TEMPLATE.substitute(
                source=source, date=date, content=content, question=question
            ),
        }],
    }
```

这个领域里最贵的 bug，是把每次请求都会变的值（时间戳、会话 ID、今天的日期）放在系统提示词开头。差一个字节，哈希不命中，于是每次调用都要按全价写入一次。你在响应的 usage 对象里能立刻看到：`cache_read_input_tokens` 是 0，`cache_creation_input_tokens` 每次请求都非零。如果你没法把这个变量挪到用户消息里，就干脆从提示词里删掉，改用工具调用传进去。

《别打断缓存》研究（Lumer 等，2026 年 1 月）在多轮基准上跑了 500 个智能体会话，测量了规范的缓存放置对一个 10,000 token 系统提示词的影响：相对于同样一条链但顺序写得随意的版本，成本降低 41% 到 80%，首 token 时间改善 13% 到 31%。[^6] 这个差距，就是“写这条链的工程师知不知道这条规则”在你账单上的样子。

反向的失效，是把完整对话历史当作每一步的前缀反复重发的那种多轮链。token 数按三角数增长：第 1 轮发 N 个 token，第 28 轮发 28N 个，30 轮累计输入大约是 N 乘以 465。一个 30 轮的智能体会话，成本可能是一次单轮调用的 50 到 200 倍，具体取决于缓存局部性和 TTL 窗口有没有把前缀保热。[^7] 修法有三种：上下文开窗（只保留最近 K 轮历史）、摘要截断（把旧轮次换成压缩摘要），或者重新插入前显式压实上下文。机制细节见[提示词缓存](../part-2-working-with-model-apis/07-prompt-caching.md)。这里要说的是：链的成本模型在任何用户敲下第一个字之前，就已经由前缀结构定死了，所以设计这个结构的地方，和你设计链其余部分的地方是同一处——那个构建请求的模板函数。

## 参考文献

[^1]: Tian Pan, "Prompt Versioning in Production: The Engineering Discipline Teams Learn the Hard Way", tianpan.co, April 2026, https://tianpan.co/blog/2026-04-09-prompt-versioning-production-llm
[^2]: Mirascope team, "4 Best Prompt Management Systems for LLM Developers in 2025", mirascope.com, 2025, https://mirascope.com/blog/prompt-management-system
[^3]: Anthropic, "Prompting best practices: Chain complex prompts", docs.anthropic.com, accessed June 2026, https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices
[^4]: Tian Pan, "The LLM Pipeline Monolith vs. Chain Trade-off: When Task Decomposition Helps and When It Hurts", tianpan.co, April 2026, https://tianpan.co/blog/2026-04-18-llm-pipeline-monolith-vs-chain
[^5]: Lennart Meincke, Ethan Mollick, Lilach Mollick, Dan Shapiro, "Prompting Science Report 2: The Decreasing Value of Chain of Thought in Prompting", arXiv:2506.07142, Wharton Generative AI Labs, June 2025, https://arxiv.org/abs/2506.07142
[^6]: Elias Lumer, Faheem Nizar, et al., "Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks", arXiv:2601.06007, January 2026, https://arxiv.org/abs/2601.06007
[^7]: Tian Pan, "The Sliding-Window Tax: Why a 30-Turn Conversation Costs More Than 30x a Single Turn", tianpan.co, May 2026, https://tianpan.co/blog/2026-05-13-sliding-window-tax-long-conversations-cost-more-than-tokens
