---
title: "模型全景图"
description: "把任何模型放进对应档位，并知道每一档是干什么的：前沿、中端、快速、本地，外加开放权重与托管、对话与推理与多模态之分。"
slug: 00-the-model-landscape
part: 1
chapter: 0
difficulty: beginner
prerequisites:
  - part-0-start-here/00-what-an-ai-engineer-is
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 模型全景图

“最好的模型”和“适合我任务的模型”不是同一句话。前者挑的是榜单第一。后者要权衡的是：从最便宜的托管模型到最贵的模型，成本差约 100 倍，而能力差距在某些基准上已经缩到 3 分以内。[^1]这两件事搞混，你就会用前沿的价格，去做一个成本只有十分之一的模型就能干好的活。

所以第一项技能不是挑模型，而是把听到的任何模型放进某个档位，并知道这个档位是干什么用的。一共四个档位。

## 四个档位

同一个请求分别走三个 Claude 模型，账单差 10 倍，措辞几乎没变：

```python
# 价格为 2026 年 6 月数据。来源：anthropic.com/pricing
PRICE_PER_MTOK = {
    "claude-fable-5":    {"in": 10.00, "out": 50.00},  # 前沿
    "claude-sonnet-4-6": {"in":  3.00, "out": 15.00},  # 中端
    "claude-haiku-4-5":  {"in":  1.00, "out":  5.00},  # 快速
}

def cost(model: str, in_tok: int, out_tok: int) -> float:
    p = PRICE_PER_MTOK[model]
    return (in_tok * p["in"] + out_tok * p["out"]) / 1_000_000

# 2,000 token 的提示词，500 token 的回答：
# claude-haiku-4-5    -> $0.0045
# claude-sonnet-4-6   -> $0.0135
# claude-fable-5      -> $0.0450
```

同一个厂商的三个模型，同一个请求，10 倍价差。把这个差距铺到整个市场，阶梯就有四级。

**前沿（frontier）**是各家实验室产品线的顶端：分数最高、价格最高、只提供托管。截至 2026 年 6 月，这一档是 Claude Fable 5（输入 $10 / 输出 $50，每百万 token）、OpenAI 的 GPT-5.5（$5 / $30），以及 Google 的 Gemini 3.1 Pro（输入 $2 / 输出 $12，预览版）。[^1][^2][^3]当质量失败的代价很高时你会伸手去拿它们：法律审阅、困难的代码生成、高风险的客户工作，或者任务真的需要深度多步推理。

**中端（mid）**用一点点能力换小得多的账单。Claude Sonnet 4.6（$3 / $15）、GPT-5.4（$2.50 / $15）、Gemini 2.5 Pro（$1.25 / $10）都在这一档。[^1][^2][^3]这是主力档，也是你做任何新功能时的默认选择。别把“中端”读成“平庸”：今天的 Sonnet 4.6 能打过 18 个月前的前沿模型；而在 MMMU-Pro 上，截至 2026 年 4 月，四个领先的前沿模型彼此只差 3 分以内。[^4]

**快速（fast）**用实打实的能力换速度和成本。这些是更小的模型，不是把前沿模型调低档。Claude Haiku 4.5（$1 / $5）、GPT-5.4 mini（$0.75 / $4.50）、Gemini 2.5 Flash（$0.30 / $2.50）在这一档，地板是 Gemini 2.5 Flash-Lite：每百万输入 token $0.10。[^1][^2][^3]Anthropic 称 Haiku 4.5 是“最快、且具备接近前沿智能的模型”。[^1]这一档用来做分类、意图识别、短摘要、简单抽取，以及 agent 内部的路由。问题在后面。

**本地（local）**指的是开放权重：实验室把真正的权重以可下载文件的形式发布，你在自己的 GPU 上跑。Llama 4 Scout 和 Maverick、Gemma 4、Qwen 3、DeepSeek V4。没有按 token 计费的账单。取而代之的是，硬件、服务栈和被叫的寻呼机都归你。

![一根四级的竖直阶梯，条形从左到右升起，每一级大约比下一级高十倍，最高的一级披着珊瑚色，按档位和价格区间标注](../../../ai-eng/part-1-how-models-behave/images/00-the-cost-ladder.png)
*每一档的价格大约是下一档的 10 倍，而它们之间的能力差距一直很小。这种不对称就是“从便宜的开始”的全部论据。*

成本阶梯每级大约爬 10 倍。能力阶梯几乎不爬。正因为有这个落差，你才应该从中端起，并且只有在测量（而不是直觉）显示差距在你的具体任务上确实要紧时才往上提。后面有一章 [成本-质量-延迟三角](./05-the-cost-quality-latency-triangle.md) 会把这种不对称变成你每次做模型决策都会用到的心智模型；本章只负责把档位立起来。

## 开放权重与托管：是运维选择，不是道德选择

前三档和第四档的分界，在于谁来跑 GPU。

- **托管（hosted）**模型藏在 API 后面。你发 token、收 token、按 token 付费，服务器、升级、宕机全归厂商。基础设施工作量是零。
- **开放权重（open-weight）**模型是你自己下载、自己服务的文件，生产环境通常用 [vLLM](https://docs.vllm.ai/)，本地开发用 Ollama。没有按 token 的收费，数据落在哪完全由你控制。

人很容易把开放权重叫成“免费”。它不是。按 token 的成本确实是零，但 GPU、网络、服务基础设施、模型更新节奏和值班时间都是真金白银；一块利用率只有 40% 的 GPU 会悄悄把你的有效单 token 成本翻倍。要跟 Gemini 2.5 Flash-Lite 这种每百万输入 token $0.10 的廉价 API 打平，你得每月推到几十亿 token 才行。[^5]而且截至 2026 年年中，开放权重模型在多数基准上落后前沿大约 3 到 6 个月。[^5]

> [!WARNING]
> **“开放权重”不等于“默认便宜”。** 如果你真要走开放权重，先租后买：先在托管推理服务商上把这个模型跑一遍，再决定要不要投入自建 GPU。*同一份*权重在不同托管商那里的价格差得很远。2026 年的一份调研显示，截至 2026 年 6 月，Llama 3.3 70B 在多家供应商之间的价格落在每百万输入 token $0.12 到 $1.05，约 9 倍差距。[^6]把这个数字当方向，别当教条；它只来自单一来源。但教训仍然成立：至少对比三家供应商，因为对开放权重模型来说，决定成本的主要是托管商，不是模型。

所以真正的决策属于运维层面，不属于伦理层面。除非下面三条中有一条成立，否则选托管：

1. **数据出不了你的围墙。** 受 HIPAA、GDPR 或内部治理约束的医疗、金融、政府数据，往往根本不能送到第三方 API。
2. **量极大。** 在一个利用率良好的集群上，一旦超过每月约 50 亿 token，托管这笔账就可能翻向开放权重一边。
3. **你必须用出不了门的数据做微调。** 不能交给厂商的专有训练数据，会逼你走本地。

这三条一条都不占，托管就在小团队关心的每一个维度上都赢。

## 对话、推理、多模态：三种模型类型，不是三个档位

档位讲的是成本和能力。类型是另一条轴，把两者混为一谈是新手常犯的错误。三种类型都跑在同一个 transformer 上；不同的是训练目标，以及模型在推理（inference）时的行为方式。

| 类型 | 它做什么 | 用来做 | 不要用来做 |
|---|---|---|---|
| **对话** | 直接生成答案 | 大多数任务；默认选择 | 没有禁区；它就是基线 |
| **推理** | 先一步步想，再回答 | 数学、代码、多步逻辑 | 实时、简单、对延迟敏感的工作 |
| **多模态** | 还能接受图像、音频或视频 | 输入不是文本的任务 | 纯文本工作（它会抬高成本） |

**对话模型**是默认项：你发一段提示词，它写一段回复。**推理模型**在给出最终答案前会生成一段隐藏的思维链，这在难题上能可测量地提升准确率。OpenAI 的 o3 解出了 Frontier Math 基准里 25% 的题，而此前没有任何模型能过 2%。[^7]Claude Sonnet 4.6 甚至自己决定什么时候该想，不用你来设预算。但思考不免费，无论 token 还是时间。后面有一章 [推理模型](./04-reasoning-models.md) 会讲测试时计算（test-time compute）的机制、隐藏的 token 成本，以及“什么时候思考才划算”的精确判据。

搞错推理的代价很具体。有个支持团队给每个请求都打开了推理，眼看着 P99 延迟从 2.4 秒飙到 41 秒；几小时内他们就回滚了。[^8]推理适用于有可验证答案的 STEM 问题、多步代码审查，以及任何“连一个合格的人也要停下来想一想”的任务。用在客服、分类和实时语音上是错的。

**多模态**模型接受非文本输入。截至 2026 年 6 月，图像输入在每一个前沿和中端模型上都是标配；原生音频和视频在 Gemini 3.x 上可用，GPT-5.5 则通过它的 Realtime API 提供。[^2][^3]图像也耗 token。在 GPT 模型里一张 512x512 的图约 210 token，所以只处理任务真正需要的模态。[^2]音频的话，先转写成文本再喂给模型，除非你真的需要实时语音。

## 合起来看

现在整张地图都在你手上了。定位任何模型只问两个问题：哪个档位（前沿、中端、快速、本地），哪种类型（对话、推理、多模态）。Claude Haiku 4.5 是快速档的托管对话模型。Llama 4 Scout 是本地的多模态对话模型。Gemini 2.5 Pro 是中端托管模型，需要时能推理。

从这张地图上落出三条默认做法，每一条都成立，直到某次测量把它推翻：

- **从中端起。** 只有当在你自己的任务上做的评测显示质量差距值这个价，才升到前沿。拿两个模型各跑一个 100 条的样本集；如果差距不到 5%，就留在中端。
- **从托管起。** 只在数据驻留、极端量级、或用出不了门的数据做微调这三种情况下才走开放权重。
- **从对话起。** 只在困难、可验证、非实时的任务上打开推理；只有输入不是文本时才动用多模态。

最后还有一个隐患值得现在就钉住，因为它在生产里真的会咬人：上面每个价格和每个模型名都带着“截至 2026 年 6 月”的标签，这不是随手加的。这份清单一年内就会过期，所以在你把某个模型选择焊进架构之前，先去厂商文档里核对当前定价。另外，在生产代码里钉死具体的模型 ID（`claude-haiku-4-5`，而不是一个会漂移的别名），这样厂商的静默更新就没法在一夜之间改变你的行为。[^1]这些数字同时也是本部分结尾你要做的基准测试的起点：在一个真实任务上，用两三个模型比成本与质量。

## 参考文献

[^1]: Anthropic，“Models overview” 与 API 定价，platform.claude.com 与 claude.com/pricing，获取于 2026 年 6 月 12 日。https://platform.claude.com/docs/en/about-claude/models/overview
[^2]: OpenAI，“API Pricing”，获取于 2026 年 6 月 12 日。https://openai.com/api/pricing/
[^3]: Google，“Gemini Developer API pricing”，最后更新于 2026 年 6 月 9 日。https://ai.google.dev/gemini-api/docs/pricing
[^4]: digitalapplied.com，“Multimodal AI Benchmarks 2026”，2026 年 4 月。https://www.digitalapplied.com/blog/multimodal-ai-benchmarks-2026-vision-audio-code
[^5]: digitalapplied.com，“Self-Hosting Open-Weight LLMs”，2026 年 5 月。https://www.digitalapplied.com/blog/self-hosting-open-weight-llms-2026-deployment-decision-guide
[^6]: saturncloud.io，“Inference Provider Comparison Report”，2026 年 5 月（单一来源；数字仅指方向）。https://saturncloud.io/reports/inference-provider-comparison-report/
[^7]: tianpan.co，“Reasoning Models: When to Use Them and When Not To”，2025 年 10 月。https://tianpan.co/blog/2025-10-10-reasoning-models-in-production
[^8]: kindatechnical.com，“When to Use Extended Thinking and When Not To”，2025-2026。https://www.kindatechnical.com/claude-ai/when-to-use-extended-thinking-and-when-not-to.html
