---
title: "引用与依据"
description: "让 RAG 的回答可以被验证：片段级引用（span-level citation）、检索为空时直接拒答，以及由忠实度与流畅度之争决定的那两件事。"
slug: 05-citations-and-grounding
part: 6
chapter: 5
difficulty: intermediate
prerequisites:
  - 00-when-you-actually-need-rag
  - 01-document-processing-and-chunking
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 引用与依据

用户问你的 RAG 助手：公司的育儿假从哪天开始。回答流畅、笃定、还带着细节：“全职员工的育儿假从入职第一天开始，最多 16 周带薪假。”可它是错的。检索器返回了三份制度文档，没有一份讲育儿假。模型是拿两年前吃进训练数据的内容在作答，语气却和一份有依据的回答毫无区别。

这正是 RAG 本该防住的失败，而且每天都在发生。接一个向量库不会让系统变得有依据，它只是让“有依据”这件事*成为可能*。模型仍然得忠实地使用检索到的上下文，你的流水线仍然得验证它确实这么做了。本章讲的就是怎么补上这一段。

一个症状底下藏着两种失败模式。第一种，模型可以引用一份真实存在的文档，而紧挨着引用的那句话是编的。第二种，检索结果为空或者答非所问时，模型照样信心十足地作答，用参数化记忆（parametric memory）把空白填满。片段级引用对付第一种，检索为空即拒答对付第二种。两者都压在同一股张力上：回答越忠实，读起来往往越不流畅，而用户偏偏更喜欢流畅的回答，哪怕它是错的。

## 片段级引用：给的是指针，不是承诺

在系统提示词里写一句“请注明来源”，产出的文字看起来像引用。模型在一句论断后面写上“[文档 2]”，你就相信这句话来自文档 2。它经常不是。模型优化的目标是通过这条指令，而不是产出可溯源的东西。你没法用字符串匹配去验证这种基于提示词的引用，因为被引的文字从来没有被抽取出来，只是被描述了一遍。

片段级引用（span-level citation）不是这样。它是一个字符区间（或页码范围），指回某份检索文档里一段精确的子串。API 把这段子串作为元数据抽出来，你可以拿它跟原文做字符串匹配。Anthropic 在 2025 年 1 月上线的 Citations API 把这件事变成了线路格式的默认行为。[^1]响应里每个文本块都带一个 `citations` 数组，每条引用给你 `cited_text`、`document_index` 和起止偏移量。被引片段不计入输出 token，在后续轮次里回传时也不计入输入 token。[^1]

```python
import anthropic

def answer_with_citations(question: str, document_text: str):
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {
                        "type": "text",
                        "media_type": "text/plain",
                        "data": document_text,
                    },
                    "title": "Source",
                    "citations": {"enabled": True},
                },
                {"type": "text", "text": question},
            ],
        }],
    )
    return response.content
```

接法就这么点。对 RAG 流水线来说，一个小而关键的细节是：每个检索到的 chunk 都应该单独作为一个 document 块，而不是拼成一个大字符串。Anthropic 的文档写得很直白：“如果你想让 Claude 能引用 RAG chunk 里的具体句子，就应该把每个 RAG chunk 放进一个纯文本文档里。”[^1]纯文本文档会按句子边界自动切块；自定义内容文档则保留你自己的切分方式，这在处理转录稿、项目符号列表或者表格时很要紧——这些地方按句子切会切成乱码。

截至 2026 年年中有一个坑：citations 和 Structured Outputs 互斥。两个都开，API 返回 400。如果你需要带引用的 JSON 结构响应，就走两趟：一趟结构化调用抽字段，一趟 citations 调用挂证据。除 Haiku 3 之外，所有在役的 Claude 模型都支持这个 API。[^1]

引用 API 不会让你的回答变成真的。它保证的是*指针*有效，不是被引片段真的支持那句论断。模型仍然可以把一条引用挂到被引片段并不蕴含的句子上。要验证这一点就得做忠实度检查，我们下面会讲到。

## 在模型有机会编造之前就拒答

检索什么都没返回时，一个经过指令微调的模型还是想帮忙。“乐于助人”这个先验在后期训练里被几十万组偏好对反复强化过，它压过系统提示词里那一行“不知道就拒答”是常态。所以最便宜、最可靠的办法，是根本不给模型这个机会。

如果检索器在相似度阈值以上返回零个 chunk，那就在调用模型之前直接返回拒答话术。它是确定性的，不花钱，而且掐掉了 RAG 流水线里最强的一个自信编造来源。

```python
import anthropic

REFUSAL = "I cannot answer this from the available documents."

REFUSAL_SYSTEM = (
    "Answer only from the provided documents. If none are relevant, "
    f'reply with exactly: "{REFUSAL}"'
)

def rag_with_refusal(question: str, retrieved_chunks: list[str]) -> str:
    if not retrieved_chunks:
        return REFUSAL

    content = [
        {
            "type": "document",
            "source": {"type": "text", "media_type": "text/plain", "data": chunk},
            "title": f"Document {i + 1}",
            "citations": {"enabled": True},
        }
        for i, chunk in enumerate(retrieved_chunks)
    ]
    content.append({"type": "text", "text": question})

    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=512,
        system=REFUSAL_SYSTEM,
        messages=[{"role": "user", "content": content}],
    )
    return "".join(b.text for b in response.content if b.type == "text")
```

快路径处理空结果。系统提示词处理更难的那种情况：检索有结果，但没有一个 chunk 真正回答了问题。模型真正吃力的是第二种。RefusalBench 是 2025 年的一项评测，覆盖 30 多个前沿模型、176 种扰动策略，结论是没有任何模型能同时在“该答时答”和“该拒时拒”两项上超过 80%。[^2]单文档拒答准确率最高的是 73%（Claude-4-Sonnet）。到了多文档任务——一个 chunk 切题、其余是嘈杂的干扰项——同一个模型掉到 36%，仅仅加了干扰项就掉了 37 个百分点。[^2]

RefusalBench 里有两条结论值得记住。第一，把模型做大、加上扩展推理，帮助都不明显；选择性拒答是“一种可训练、对对齐敏感的能力”，不是规模大了就会自己长出来的能力。[^2]第二，同一基准上 GPT-4o 对可回答的问题有 62.8% 的时间在拒答，而该拒未拒只占 4.3%。这不算更好的结果。一个把三分之二正常问题都拒掉的系统是废的。过度拒答是对称的失败模式，它摧毁用户信任的速度和自信幻觉一样快。

实际含义是：两个方向都要盯。记录系统的拒答率，同时跑一套已知可回答的评测集来抓过度拒答。如果改了一版提示词之后拒答率往上飘，用户会比你的看板更早发现。

在拒答准确率真正要紧的领域（医疗、法律、金融），光靠系统提示词的指令不够。基于训练的方法确实做得更好。2025 年 6 月的 Ground-GRPO 论文报告：在 8B 参数的开源权重模型上，两阶段训练（先只用可回答问题，再混入不可回答的样本）比单阶段训练把有依据拒答的 F1 提高了 15.8 个点。[^3]这就是升级路径：做原型用提示词，诊断问题用评测，只靠提示词撞到天花板时再上微调。

## 忠实度不等于准确率

这两个词在产品评审里经常被当成同义词用，但不该这样。忠实度（faithfulness）问的是：回答里的每一条论断都能追溯到检索到的上下文吗？准确率（accuracy）问的是：这个回答本身是真的吗？一个回答可以既忠实又错误（文档过时了，你忠实地引用了一条过期事实）。它也可以既准确又不忠实（模型从训练数据里想起了正确答案，引用只是装饰）。把两者搞混，正是团队把“有依据”的助手上线、然后自信地误导用户的原因。

![一张 2x2 网格，展示高/低忠实度与高/低准确率的四种组合，右上角的目标象限用珊瑚色高亮](../../../ai-eng/part-6-retrieval-augmented-generation/images/05-faithfulness-vs-accuracy.png)
*忠实度和准确率是两条独立的维度。右上象限是目标；右下象限是检索来源本身有错时，引用机制制造出来的失败模式。*

忠实度是你能低成本测量的指标，因为它不需要标准答案。把回答拆成原子论断，用 NLI 模型或 LLM 裁判逐条对照检索到的上下文，再用被支持的条数除以总数。这就是 RAGAS 的忠实度分数，只要你愿意，它可以对每一条线上响应都跑一遍。[^4]

```python
import anthropic, json

DECOMPOSE = """Extract atomic claims from this answer as a JSON list.
Answer: {answer}
Return only valid JSON like ["claim1", "claim2"]."""

CHECK = """Does the context support the claim?
Context: {context}
Claim: {claim}
Reply with exactly "yes" or "no"."""

def faithfulness_score(answer: str, context: str) -> float:
    client = anthropic.Anthropic()
    decomp = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=512,
        messages=[{"role": "user", "content": DECOMPOSE.format(answer=answer)}],
    )
    claims = json.loads(decomp.content[0].text)
    if not claims:
        return 0.0

    supported = 0
    for claim in claims:
        check = client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=8,
            messages=[{
                "role": "user",
                "content": CHECK.format(context=context, claim=claim),
            }],
        )
        if "yes" in check.content[0].text.lower():
            supported += 1
    return supported / len(claims)
```

线上环境里，把逐条论断的 LLM 调用换成一个小 NLI 模型（MiniCheck 之类又便宜又快，跑在请求链路上完全够）。卡在哪个阈值是业务决策：法律和医疗系统常常要求 0.9 以上，通用问答 0.7 就能接受。低于阈值就退回到拒答话术，别把一条无依据的回答发出去。

## 靠提示词拔不出来的那场拔河

就算引用流水线做得完美，还有一个更深的问题：当检索到的文档和模型的参数化记忆冲突时，模型必须选一边。而它的选择是有规律的——不是按哪边正确来选，而是按它*看到文档之前*有多确定来选。

斯坦福的 Wu、Wu 和 Zou 在 2024 年做了这个实验最干净的一个版本。[^5]他们取六个领域（药物剂量、体育纪录、新闻事实、日期、人名、地点）共 1,294 个问答对，先在不给检索的情况下测出 GPT-4 对每个问题的先验概率，然后注入检索文档（有正确的，也有故意扰动的），观察模型是跟着文档走还是跟着先验走。以下是 `gpt-4-turbo-preview`（2024 年 3 月访问）上的主要数字：

- 检索内容正确时：准确率 94%，而同样的问题不给检索时是 34.7%。
- 所有条件合并看：斜率为 -0.23。模型的先验置信度每提高 10%，它跟随检索文档的频率就下降 2.3%。

第二个数字一行就说清了生产环境的风险：模型在训练中“最笃定”的那些问题，恰恰是它最愿意无视你检索文档的那些问题。严格的提示词（“只依据提供的文档回答，不要使用任何其他知识”）能把服从度整体拉高，但压不平这条斜率。[^5]宽松的提示词会让斜率更陡。这个机制扛得住提示词工程。

对流畅度的含义很直接。一个能在多份文档之间平滑综合、再用训练数据补上文体黏合剂的模型，读起来比只会照抄片段的模型自然得多。即便流畅的回答里包含无依据的论断，用户在短期评分里还是给它更高的分。所以如果你优化用户满意度却不测忠实度，你就是在往不忠实的方向优化。忠实度闸门存在的理由，正是那个显而易见的用户侧指标会把你往反方向拽。

## 能自洽运转的一条链路

本章的三个机制组合成一条生产路径：

1. 调用模型之前，先看检索结果。阈值以上零个 chunk，立刻返回拒答话术，不花一分 LLM 的钱。
2. 调用模型时，打开片段级引用，把每个 chunk 各自作为一个 document 块传进去。响应里带回来的指针你可以做字符串匹配。
3. 拿到响应之后，对“回答 + 上下文”这一对打忠实度分。低于你的领域阈值就退回到拒答话术。无论如何都把这个分数记下来。

每一层拦的是一类不同的失败。检索守卫拦空结果。引用 API 拦“我引了文档 2”而根本没有文档 2 支撑的论断。忠实度检查拦最难的一类：一个自信的回答，引了一份真实文档，而被引片段其实并不支持它。少任何一层，它负责的那类失败就直接漏到用户面前。

本章没给你的，是判断检索一开始有没有捞到正确文档的能力。在错的来源上打出 0.95 的忠实度，就是一个带着漂亮收据、自信错误的回答。那正是图中右下象限，唯一的解法在上游：更好的检索、更好的来源审查，以及 [评测检索](../../../ai-eng/part-6-retrieval-augmented-generation/07-evaluating-retrieval.md) 里讲的那套检索评测纪律。依据机制让回答可被验证；拿好来源去验证回答，才让回答是对的。

放到架构尺度看，[RAG 系统的 HLD 章节](../../hld/part-9-ai-ml-system-design/01-rag-pipelines.md)讲的是白板上的那一版：引用存储放在哪、忠实度分数怎么喂给质量闸门、拒答率看板怎么接到事故响应里。

## 参考文献

[^1]: Anthropic, "Citations," Claude API Docs, https://docs.anthropic.com/en/docs/build-with-claude/citations (accessed June 2026).

[^2]: Aashiq Muhamed, Leonardo F. R. Ribeiro, Markus Dreyer, Virginia Smith, Mona T. Diab, "RefusalBench: Generative Evaluation of Selective Refusal in Grounded Language Models," Carnegie Mellon University and Amazon AGI, arXiv:2510.10390, April 2025. https://arxiv.org/html/2510.10390v1

[^3]: Shang Hong Sim et al., "Lessons from Training Grounded LLMs with Verifiable Rewards," Singapore Management University and DSO National Laboratories, arXiv:2506.15522, June 2025. https://arxiv.org/html/2506.15522v1

[^4]: Shahul Es, Jithin James, Luis Espinosa-Anke, Steven Schockaert, "RAGAS: Automated Evaluation of Retrieval Augmented Generation," arXiv:2309.15217, 2023. https://arxiv.org/abs/2309.15217

[^5]: Kevin Wu, Eric Wu, James Zou, "How faithful are RAG models? Quantifying the tug-of-war between RAG and LLMs' internal prior," Stanford University, arXiv:2404.10198, April 2024 (updated February 2025). https://arxiv.org/abs/2404.10198
