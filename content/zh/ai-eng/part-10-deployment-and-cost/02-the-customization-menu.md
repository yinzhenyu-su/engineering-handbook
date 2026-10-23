---
title: "定制化菜单"
description: "把微调、LoRA 和蒸馏当作决策而不是技术：什么时候该选哪一个、代价是什么，以及那个让坏系统看起来健康的污染陷阱。"
slug: 02-the-customization-menu
part: 10
chapter: 2
difficulty: intermediate
prerequisites:
  - part-0-start-here/03-the-escalation-ladder
  - part-4-eval-fundamentals/02-your-first-eval-set
  - part-6-retrieval-augmented-generation/00-when-you-actually-need-rag
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 定制化菜单

一个做客服工单分类的团队发现，用提示词驱动的 GPT-5 模型准确率卡在 87% 上不去。他们想要 92%。技术负责人说“上微调吧”，于是六周、一万八千美元，在内部流量上搭了一套 LoRA 流水线。微调后的模型在他们的评测集上拿到 94 分。上线。新用户开始抱怨。评测集与线上表现的差距有二十个点。

出问题的地方有个名字：污染（contamination）。微调数据和评测集来自同一个查询池。模型没有变聪明，它背下了考题。

这一章讲三个工具——监督微调、LoRA、蒸馏——但讲的是怎么选，不是怎么做。执行层面的细节（训练循环、超参数、优化器选择）属于你的服务商文档，也属于那些全职做 ML 工程的人。作为 AI 工程师，你需要的是决策框架：什么时候该选哪一个、代价是多少、以及哪些失效模式咬人最狠。

## 升级顺序

[升级阶梯](../part-0-start-here/03-the-escalation-ladder.md)在 Part 0 就定下了顺序：提示词、上下文、检索、工具、智能体、微调。微调排在最后一格，因为它在算力、时间、数据和长期维护上最贵，也因为对多数任务来说，提示词工程能做到微调质量的八成左右。[^1] 这个八成是经验数字，不是保证。做钓鱼邮件检测，微调比提示词高 10 个 F1 点。某些临床分类任务上，提示词和微调的差距落在噪声范围内。不把两边都测一遍，你没法预判自己的任务落在哪一侧。

在伸手去够任何一个定制化工具之前，第一个要判断的是：这是哪种缺口？

- **知识缺口。**“模型不知道我们上个季度上了什么产品更新。”这是检索问题。加上 [RAG](../part-6-retrieval-augmented-generation/00-when-you-actually-need-rag.md)，不要微调。Ovadia 等人（EMNLP 2024）证明，在注入事实性知识这件事上，RAG 稳定胜过无监督微调，而微调模型会随着你的数据一起过期。[^2]
- **行为缺口。**“模型输出的格式不对”，或者“不按人设说话”，或者“没法稳定遵守我们的 schema”。这才是微调值钱的地方。RAG 修不了输出结构。
- **两者都有。**行为靠微调，知识靠叠一层 RAG。两者可以组合。

如果缺口是知识：别往下读了，去修检索。本章剩下部分讲的是行为缺口，以及那些 API 经济性确实不成立的小部分场景。

## 三个工具，三个决策

![一棵纵向决策树：从“质量不达标”出发，按缺口类型和经济性分支，最终落到四种技术，越便宜的选项越靠上，读者一眼能看出这条攀爬路径。](../../../ai-eng/part-10-deployment-and-cost/images/02-customization-decision-tree.png)
*把定制化菜单画成决策树。最便宜的答案在上，最贵的在下；只有上一格确实失败了，才往下走一格。*

**监督微调（SFT，supervised fine-tuning）**用提示词—回复对构成的数据集，在所有模型参数上继续做基于梯度的优化。每一层的每一个权重都会被更新。换来的是最深的行为适配，代价也成比例：7B 模型做完整 SFT 需要 100-120 GB 显存；70B 即使在混合精度下也需要多节点 H100 集群。[^1] LIMA（Zhou 等人，NeurIPS 2023）证明 1000 条精心筛选的样本就能匹敌用十万条以上样本训练的模型；结论是数据质量压过数据数量，不是微调便宜。[^3]

**LoRA**（低秩适配，Low-Rank Adaptation，Hu 等人 2021）冻结预训练权重，改为训练小的秩分解矩阵，在推理服务时再合并回权重里。[^4] 对 GPT-3 175B、秩取 4 的情况，LoRA 把可训练参数减少了一万倍，GPU 显存比完整 SFT 少 3 倍。推理延迟不增加，因为适配器在提供服务之前已经折进基础权重了。

Biderman 等人发表在 TMLR（2024 年 8 月）的论文是这项对比的定论：LoRA 学得更少，也忘得更少。[^5] 在深度领域适配上（比如用 200 亿 token 学一个新代码库的风格），LoRA 表现不佳，因为要装下这次更新所需的秩，是大家通常配置的 10-100 倍。而在用几千条高质量样本做行为适配上，LoRA 能追平或接近完整 SFT，并且在目标任务之外更好地保留基础模型的能力。2026 年 3 月一项对照实验测到：LoRA 下灾难性遗忘为 0.6%，完整 SFT 下为 19.9%（配对 t 检验，p=0.002）。[^6]

**蒸馏（distillation）**用在 LLM 上，操作上很简单：让一个大模型当老师，为你的任务生成数据集，再用这个数据集微调一个更小的学生模型。Hsieh 等人（“Distilling Step-by-Step”，ACL 2023）证明，一个用思维链推理过程蒸馏出来的 770M T5 模型，只用了 80% 的数据，就打赢了少样本提示的 540B PaLM。[^7] 学生并不比老师聪明；它只是为某一个窄任务压出来的一条捷径。这既是它的威力，也是它的上限。

由此得出三条规则：

- **在 LoRA 和完整 SFT 之间选 LoRA**，除非你确实要在几十亿 token 上做深度领域适配。多数行为适配场景下，LoRA 更快、更便宜、忘得更少。
- **不要从弱老师那里蒸馏。**如果老师在你的任务上最高只有 70% 准确率，学生就一定低于 70%。先去修老师的提示词。
- **蒸馏是为了部署约束，不是为了省 API 钱。**Apple Intelligence 在手机 NPU 上跑蒸馏模型，是因为物理上没有别的可能。多数团队没有这种约束，而前沿 API 的价格在 2024-2026 年间跌了大约八成，这直接压垮了按 token 省钱的论证。[^8]

## 决定成败的盈亏平衡计算

在开任何微调项目之前，先把账算一遍。盈亏平衡月数等于训练成本除以每月推理省下的钱。完整训练成本包括数据准备（占项目总成本的 20-40%）、MLOps 工程人力（做过这事的人，全成本 17 万到 21.5 万美元），以及基础模型更新后一轮又一轮的重训。[^1]

```python
def break_even_months(
    training_cost_usd: float,
    monthly_volume_m_tokens: float,
    saving_per_m_tokens: float,
) -> float:
    """Break-even = total training cost / monthly inference savings."""
    monthly_savings = monthly_volume_m_tokens * saving_per_m_tokens
    return training_cost_usd / monthly_savings if monthly_savings > 0 else float("inf")

# 例：训练 2 万美元，每月 1000 万 token，每百万 token 省 1 美元 -> 24 个月
print(f"{break_even_months(20_000, 10, 1.0):.0f} months")
```

决策规则：盈亏平衡超过 6 个月，就不要微调。超过 12 个月，蒸馏多半也是错的；留在 API 上。[^1] 这些数字背后是一个不浪漫的事实：前沿模型的价格一直在跌，基础模型一直在变强，而今天训出来的微调模型从上线那天起就在贬值。

## 污染陷阱

接下来是最咬人的部分。

污染指的是评测样本（或高度接近的改写）混进了训练集。模型背下了评测，报出来的分数很好看，线上质量一点没变。之后每一次评测都变成记忆测验，团队带着并不属于他们的信心上线。

它看不见。它很常见。而且一旦发生，结构上就很难补救。

公开榜单的世界已经记录了两年。SWE-bench Verified 显示出 10.6% 的泄漏率；OpenAI 在 2025 年底公开停止报告该项数据，因为内部审计发现每个主流前沿模型都能对某些任务逐字复现标准补丁。[^9] 论文《Inference-Time Decontamination》（arXiv:2601.19334，2026）发现，在正确控制污染之后，GSM8K 上的准确率下降约 22.9%，MMLU 上下降 19.0%。一个在 SWE-bench Verified 上拿 93.9% 的前沿模型，在做了污染控制的 SWE-bench Pro 上只有 45.9%。差距 48 个点。

放到你自己的微调项目上，污染通常不是恶意造成的，而是通过完全合规的流程进来的。你在第一个月建的评测集来自用户查询。你在第三个月拼的微调集也来自用户查询。如果切分没有在采集当时被强制执行，事后就没有可靠办法查出重叠。

```python
import hashlib

def ngram_hashes(text: str, n: int = 13) -> set[str]:
    tokens = text.split()
    return {
        hashlib.md5(" ".join(tokens[i:i+n]).encode()).hexdigest()
        for i in range(max(0, len(tokens) - n + 1))
    }

def contamination_rate(eval_examples: list[str], training_corpus: list[str], n: int = 13) -> float:
    """Fraction of eval examples with at least one 13-gram match in the training corpus.
    Above ~5% warrants a full audit. (GPT-3 used this method.)
    """
    corpus_hashes: set[str] = set()
    for doc in training_corpus:
        corpus_hashes.update(ngram_hashes(doc, n))
    contaminated = sum(1 for ex in eval_examples if ngram_hashes(ex, n) & corpus_hashes)
    return contaminated / len(eval_examples) if eval_examples else 0.0
```

这只是个初步探针，不是保证。13-gram 哈希正是 GPT-3 训练流水线用的办法。[^9] 如果你看到重叠超过 5%，就做一次完整审计。更好的做法是从结构上防止污染。

防污染的规则是组织性的，不是技术性的：

- **在开始采集任何微调数据之前，先把测试集划出来。**单独存放，加访问控制。绝不把这些样本用作训练数据，也不用作合成数据生成的种子提示。
- **留一个孪生留出集。**一个分布和难度都相同的私有切分。干净的模型在公开评测集上的分数落在统计噪声范围内；被污染的模型在公开切分上会系统性偏高。
- **做合成数据时，用没有参与构建评测集的留出生产流量去提示老师模型。**否则合成数据的分布与现实不匹配，学生在真实用户身上会退化。

LiveCodeBench 和 LiveBench 解决公开榜单版本的这个问题，靠的是给每道题打发布日期，并且只用模型训练截止之后发布的题来评测。SWE-bench Pro 用的是私有的 GPL 授权仓库。模式是一样的：用结构性屏障替代组织性信任。[^9]

## 买还是自建

如果你确实决定微调，买还是自建这个问题，基本由你已经在哪儿跑决定。截至 2026 年 6 月：

- **Google Vertex AI** 对 Gemini 2.5 Flash 的监督微调收每百万训练 token 5 美元，Gemini 2.5 Pro 收 25 美元。Gemini 2.5 及更早的模型，微调后的推理价格与基础模型相同；更新的模型上这个规则会变。[^10]
- **OpenAI** 正在关停自助式微调平台。该平台已对新用户不可用。o4-mini 的强化微调按训练时间每小时 100 美元计费，推理输入每百万 token 4 美元。[^11]
- **自建 LoRA**（用 HuggingFace PEFT）省掉了按 token 的训练费，但需要 GPU 基础设施、实验追踪和服务化。为一个能胜任的工程师预留每年 17 万到 21.5 万美元。[^1]

规律是：在中低用量下，托管微调 API 在运营成本上更划算，哪怕单价看起来高。在高用量且任务稳定的情况下，自建 LoRA 加自建基础模型才是终点，但前提是你已经证明用量是真的、任务定义是稳的。

## 说到底，你该站在哪里

多数团队不该微调。提示词工程比他们预期更快就被榨干，RAG 负责知识缺口，盈亏平衡的账算不过来。真正该微调的团队（Harvey AI，手里有 100 亿 token 以上的美国判例法，任何预训练语料里都没有；Apple，要在手机 NPU 上部署）都有一个强制理由：一份确实在预训练之外的语料，或者一个排除了云端推理的部署约束。[^1]

如果你在考虑微调，按顺序过关。缺口是行为还是知识？如果是知识，去修检索。如果是行为，提示词是否已经榨干？如果是，盈亏平衡是否落在 6 个月以内？如果是，你能保证微调数据与评测集干净切分吗？四关里有一关答不上“是”，你就还没准备好微调。

## 参考文献

[^1]: Tian Pan, "Fine-Tuning Is Usually the Wrong Move: A Decision Framework for LLM Customization", TianPan.co, April 2026, https://tianpan.co/blog/2025-11-06-fine-tuning-vs-prompt-engineering-decision-framework
[^2]: Oded Ovadia et al., "Fine-Tuning or Retrieval? Comparing Knowledge Injection in LLMs", EMNLP 2024, arXiv:2312.05934, https://arxiv.org/abs/2312.05934
[^3]: Chunting Zhou et al., "LIMA: Less Is More for Alignment", NeurIPS 2023, arXiv:2305.11206, https://arxiv.org/abs/2305.11206
[^4]: Edward Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models", arXiv:2106.09685, October 2021, https://arxiv.org/abs/2106.09685
[^5]: Dan Biderman et al., "LoRA Learns Less and Forgets Less", Transactions on Machine Learning Research, August 2024, arXiv:2405.09673, https://arxiv.org/abs/2405.09673
[^6]: arXiv:2603.27707, "Catastrophic Forgetting in Sequential Fine-tuning", March 2026
[^7]: Cheng-Yu Hsieh et al., "Distilling Step-by-Step!", ACL Findings 2023, arXiv:2305.02301, https://arxiv.org/abs/2305.02301
[^8]: Tian Pan, "Knowledge Distillation Economics: When Compressing a Frontier Model Actually Pays Off", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-09-knowledge-distillation-economics-production-ai
[^9]: Tian Pan, "The Benchmark Leak: How Your Eval Set Quietly Joins the Training Corpus", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-23-benchmark-leak-eval-contamination
[^10]: Google, "Vertex AI Generative AI pricing", June 2026, https://cloud.google.com/vertex-ai/generative-ai/pricing
[^11]: OpenAI, "Developer pricing", June 2026, https://developers.openai.com/api/docs/pricing
