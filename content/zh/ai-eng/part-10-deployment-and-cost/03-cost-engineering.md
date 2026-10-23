---
title: "成本工程"
description: "五个相互正交的杠杆（缓存、批处理、路由、量化、蒸馏）以乘性方式叠加，在不改变模型质量的前提下，把五位数的月账单砍掉约 80%。"
slug: 03-cost-engineering
part: 10
chapter: 3
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
  - part-2-working-with-model-apis/07-prompt-caching
  - part-2-working-with-model-apis/06-gateways-and-routing
  - 02-the-customization-menu
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 成本工程

一个 SaaS 产品每月用 Claude Sonnet 4.6 跑 100 万条客服工单摘要，账单是每月 19,500 美元。只上缓存和批处理——纯配置改动，不训练，也不掉质量——就能把它压到大约 6,200 美元。为负载里最窄的那一块训练一个蒸馏学生模型，可以进一步降到约 5,000 美元；更重要的是，它把剩下最大的一笔开销变成了固定的 GPU 租金，不再随流量上涨。

本章讲的就是这笔账。每个杠杆打的是账单的不同部位，而且它们之间是相乘关系。做对一个省 50%，五个都做对，账单掉四分之三，剩下那块顽固的部分是平的，不再线性增长。

## 账单上只有一项计费

每一笔 API 费用都是同一个算式：处理的 token 数乘以每 token 价格。五个杠杆不改变这一行，它们改变的是实际单价（effective price per token）——把工作导向更便宜的路径。

| 杠杆 | 改变了什么 | 典型节省 |
|---|---|---|
| 提示缓存（prompt caching） | 缓存命中的 token 按基础价的 10% 计费 | 可缓存的输入省 30-70% |
| 批处理 | 异步 token 打五折 | 适合批处理的工作省 50% |
| 模型路由 | 简单查询走更便宜的档位 | 被路由的流量省 30-85% |
| 量化 | 自托管权重用更低精度 | GPU 服务成本降 2-7 倍 |
| 蒸馏 | 窄任务交给 7B 学生模型 | 稳定的窄任务省 25-130 倍 |

顺序很重要：最容易见效的放前面，投入最大的放最后。如果你从没优化过 LLM 账单，前两个杠杆通常占全部节省的 60-70%，而且一周之内就能上线。

## 杠杆 1：提示缓存

每次 LLM 调用都是无状态的。你发过去的每个 token，服务商都要重跑一遍 prefill（完整的 KV 计算）。提示缓存把稳定前缀的 KV 张量存下来，后续请求直接复用，按基础输入价的约 10% 计费。折扣力度各家一致，实现方式各不相同。

以 2026 年 6 月的 Claude Sonnet 4.6 为例：基础输入 $3.00/MTok，缓存读取 $0.30/MTok，5 分钟 TTL 的缓存写入 $3.75/MTok（基础价的 1.25 倍），1 小时 TTL $6.00/MTok（基础价的 2.0 倍）。[^1] Anthropic 要求你用 `cache_control` 主动标记可缓存的内容块。OpenAI 的缓存对超过 1,024 token 的前缀全自动生效，缓存输入打一折（90% 折扣）。[^2] Google Gemini 需要显式创建 `cachedContent` 资源，另外按每 MTok 每小时收存储费。[^3]

回本很快。以 Anthropic 的 5 分钟 TTL 为例：`(3.75 - 3.00) / (3.00 - 0.30) = 0.28`，意思是写入之后只要命中一次缓存，多付的钱就回来了。一次都没命中，这笔钱就白花；超过一次，几乎全是净赚。

这个坑每个团队都踩过：

> [!WARNING]
> **盯住 API 响应里的 `cache_creation_input_tokens` 和 `cache_read_input_tokens`。** 如果每次请求的 creation 都非零，而 read 一直贴着零，说明缓存从来没命中过。原因几乎总是：某个时间戳、用户 ID 或每次请求都变的值混进了被缓存的块里，把前缀哈希打散了。把所有动态内容挪到缓存断点*之后*。

各服务商的具体机制，见 [提示缓存](../part-2-working-with-model-apis/07-prompt-caching.md)。本章的决策规则是：稳定前缀至少 1,024 token，并且在 TTL 窗口内会被多次请求复用，那就缓存它。

## 杠杆 2：批处理

三家主要服务商都提供异步批处理 API：输入和输出都打五折，24 小时周转，模型一样，质量一样。[^4] Anthropic 的 Message Batches API 每批最多 100,000 个请求或 256 MB；结果在队列里保留 29 天。尽管 SLA 写的是 24 小时，大多数批次一小时内就跑完了。

决策只有一个：用户等得起吗？等得起就批处理。夜间摘要、评测流水线、文档处理、每日摘要任务、内部分析、RAG 灌库，全都天然适合。实时对话、agent 工具调用，以及任何面向用户的场景都不适合。

批处理和缓存能干净地叠加。输入里被缓存的部分按 10% 费率计费，批处理折扣再叠上去。批处理请求里命中的缓存 token，按 0.10 × 0.50 = 标准价的 5% 计费。[^4] 批处理负载要用 1 小时的缓存 TTL；默认的 5 分钟在异步处理通常完成之前就过期了。

## 杠杆 3：模型路由

不是每个请求都需要前沿模型。一个问“你们营业时间是什么”的工单用不着 GPT-5.5，一个复杂的多步排查请求才需要。模型路由给每个查询分类，派发给能满足你质量下限的最便宜档位，前沿模型留作兜底。

UC Berkeley 的 RouteLLM 论文（Ong 等，2024）用基于 Chatbot Arena 偏好数据训练的矩阵分解路由器，在 MT Bench 上把成本降低 85%，同时保住 GPT-4 质量的 95%。[^5] 2026 年的闭环路由论文（arXiv:2604.23577）把它扩展到生产负载上的多类路由，成本降 40-85%，结构化任务上的质量保持在 96-100%。[^6]

截至 2026 年 6 月，档位之间的价差决定了杠杆的上限：

- **Claude Haiku 4.5 vs Sonnet 4.6：** 每 MTok 输入/输出 $1.00/$5.00 vs $3.00/$15.00，差 3 倍
- **GPT-5.4-mini vs GPT-5.5：** $0.75/$4.50 vs $5.00/$30.00，差 6.7 倍
- **Gemini 3.1 Flash-Lite vs 3.5 Flash：** $0.25/$1.50 vs $1.50/$9.00，差 6 倍

三种实现风格，按复杂度排序：

1. **基于规则的路由。** 按请求类型、token 长度或业务标记分类。零 ML 开销，最容易审计。查询类型定义清楚时，这是该选的默认做法。
2. **基于分类器的路由。** 训一个小模型（逻辑回归，或微调过的 BERT 级编码器）预测便宜档位能否满足质量要求。需要带标注的评测集。
3. **级联。** 先发给便宜模型，判定输出，不达标再升级。不需要预先训练路由，但升级路径会增加延迟。

坑在质量悬崖上。靠近路由阈值的查询，质量方差最大；而能力下降往往表现为“看起来很自信的错误答案”，用户会照着它去操作。跑一次影子评测：对 1-5% 被路由的请求同时调用前沿模型，用 LLM 裁判做对比。阈值按质量数据调，不按成本目标调。裁判机制见 [LLM-as-judge](../part-8-evaluation-and-observability/01-llm-as-judge.md)。

另一个值得知道的相互作用：路由会破坏缓存，除非你按档位分别缓存。为 Sonnet 缓存的前缀不能在 Haiku 上复用，哪怕字节完全一样。在每个请求上配好同样的 `cache_control` 断点，并接受按档位的写入成本。

## 杠杆 4：量化（仅自托管）

量化把模型权重的精度从 16 位 BF16 降到 8 位（FP8 或 INT8）或 4 位（W4A16 INT4）。这样能压缩 GPU 显存，让更大的模型跑在更小的硬件上，也能在同一套集群上多服务几路并发请求。

最彻底的一项研究是 Kurtic 等（Red Hat AI，arXiv 2411.02355），在 vLLM 0.6.4 上针对 A6000/A100/H100 评估了 Llama 3.1 8B/70B/405B 的三种格式：[^7]

- **W8A8-FP（FP8）。** 实践中无损。需要 Hopper 或 Ada Lovelace 硬件。最适合高吞吐的异步部署。相比 BF16 成本降 1.84-3.17 倍。
- **W8A8-INT（INT8）。** 平均损失 1-3% 精度。70B 以上的模型需要 SmoothQuant。
- **W4A16-INT（INT4 权重，FP16 激活）。** 最适合对延迟敏感的同步部署。8B/70B 成本降 2-3 倍；405B 降 5-7 倍（4 张 A100 取代 16 张）。

一句话规则：同步服务用 W4A16-INT，异步批处理用 W8A8-FP，买得起 H100 就上 FP8。

这个杠杆只在你用自己的 GPU 服务开放权重模型时才成立。托管 API 内部自己管量化，你手上没有旋钮。下面的算例里，这个杠杆对 Claude API 负载的贡献是零；但如果你已经把大批流量迁到自托管的学生模型上，它就是主要的降本手段。

## 杠杆 5：蒸馏

蒸馏训练一个更小的学生模型，让它在窄任务上复现大模型（教师）的输出。机制和决策框架见 [定制化菜单](02-the-customization-menu.md)；本章只把蒸馏当成本杠杆，只谈 ROI。

Amazon 的 PGKD 论文（arXiv 2411.05045）在窄文本分类任务上做到比前沿教师模型快 130 倍推理、低 25 倍成本。[^8] 这些数字是真的，但只对特定任务成立：分类，不是通用生成。通用负载上收益更小，而且很大程度上取决于你有没有先把教师的提示词调好。

2024 到 2026 年，经济性发生了剧烈变化。前沿 API 价格跌了约 80%，按 token 省钱的论点因此被削弱。今天蒸馏最有说服力的场景：

- **100 毫秒以内的延迟**，托管 API 谁也做不到
- **隐私受限的部署**（数据不能出私网）
- **数千万次请求**的稳定窄任务，每月 3,000 美元的 7B 学生模型胜过每月 20,000 美元的前沿 API 调用

盈亏平衡公式：训练成本除以每月推理节省。从零训一个 7B 学生模型，光算力就要 5,000-30,000 美元，人工还没算。如果每月只省 500 美元，回本期是 10-60 个月。别做。[^9]

## 完整算例

现在把所有东西叠到那份每月 19,500 美元的负载上。前提：

- 每月 100 万次请求
- 每次请求输入 4,500 token（4,000 稳定系统提示 + 500 工单相关内容）
- 每次请求输出 400 token（JSON 摘要）
- 基础模型：Claude Sonnet 4.6（每 MTok 输入 $3.00 / 输出 $15.00）

```text
Baseline:
  input:  1,000,000 x 4,500 tokens x $3.00/MTok  = $13,500
  output: 1,000,000 x 400   tokens x $15.00/MTok = $6,000
  total                                          = $19,500/month
```

![一张水平瀑布图：从 19,500 美元的基线出发，依次经过缓存、批处理、路由和蒸馏逐级下降，最终落在约 4,960 美元；每一级都标注了所用杠杆和去掉的金额。](../../../ai-eng/part-10-deployment-and-cost/images/03-five-lever-waterfall.png)
*五个杠杆叠在同一份负载上。每一级是再应用一个杠杆后的累计账单。单级降幅最大的是提示缓存，其次是批处理；蒸馏削掉剩下的部分，并让它不再随未来的流量上涨。*

**应用杠杆 1（缓存）后：** 4,000 token 的系统提示在每个请求里完全相同。把它标记为可缓存。4,500 个输入 token 里有 4,000 个（89%）在第一次写入之后就走缓存。假设 5 分钟 TTL 刷新，每 100 次请求发生一次缓存写入：

```text
cache reads:  4,000 x $0.30/MTok x 1,000,000 = $1,200
uncached:     500   x $3.00/MTok x 1,000,000 = $1,500
cache writes: 4,000 x $3.75/MTok x 10,000    = $150
output:                                        $6,000
total                                          $8,850 (55% off baseline)
```

**应用杠杆 2（批处理）后：** 这些摘要里有 60% 是夜间摘要任务，等 24 小时没问题。这部分走 Message Batches API，输入和输出都打五折。实时部分保持全价。

```text
total after caching + batching: ~$6,230 (68% off baseline)
```

**应用杠杆 3（路由）后：** 看一遍查询分布，发现 55% 是简单抽取，Haiku 4.5 在相同评测分数下就能处理。把这块路由到 Haiku（每 MTok $1.00/$5.00），剩下 45% 留在 Sonnet 4.6。

这个场景里，路由几乎没动账单，最后落在约 6,460 美元。被路由的请求丢掉了缓存收益，因为 Haiku 的缓存和 Sonnet 的缓存是两个独立命名空间。只有当你在便宜档位上也配好缓存，或者被路由的量大到压倒性时，路由才有显著效果。

**应用杠杆 5（蒸馏）后：** 30% 的负载是窄工单分类（账单/技术/账号分流）。用教师模型的合成输出训一个 7B 学生模型。训练：一次性 15,000 美元。服务：这个量级下，蒸馏那一块平均约每秒 0.1 个请求，一张 L4 GPU 就能带余量扛住，按需实例大约每月 600 美元。折算成每 token 费率，大致相当于每 MTok $0.40，但按 token 折算会掩盖真正的性质：GPU 是一笔固定成本，token 量涨了它也不动。

```text
that 30% on Sonnet (with cache + batch) would cost:   ~$1,870
distilled portion (300K req): one L4 GPU, flat:         ~$600
remaining 700K req on Sonnet (with cache + batch):    ~$4,360
total: ~$4,960/month (75% off baseline)
```

把 15,000 美元训练成本摊到 12 个月，每月多 1,250 美元，第一年总计约 6,210 美元，只比只做缓存加批处理的账单低一点点。这是诚实的读法：在这个量级上，蒸馏是对增长下的注，不是天上掉的钱。API 杠杆的节省随账单一起长，GPU 那一行不会。量翻到三倍时，蒸馏那部分仍然约 600 美元，而等价的 Sonnet 调用要每月约 5,600 美元，这时这笔注才回本。训练之前先把盈亏平衡算一遍，不是训练之后。

## 杠杆不会告诉你的事

有三件事算不出来，但永远重要。

最大的错误不是选错杠杆，而是上了某个杠杆却从不测它。每个“以为在缓存、其实没缓存”的团队症状都一样：cache_creation_input_tokens 居高不下，cache_read_input_tokens 一直贴着零。上线之前先把这两个指标接进看板。面板设计见 [监控与看板](../part-8-evaluation-and-observability/07-monitoring-and-dashboards.md)。

第二件是价格烂得快。本章每个数字都截止到 2026 年中，你读到的时候已经不准了。算式不变，输入会变。把 75% 这个数字讲给别人之前，先拿当前价格把算例重跑一遍。

第三件：伤质量的降本不算降本。路由阈值按成本目标定、不经裁判验证，就会制造静默回归。不重新训练的蒸馏学生会漂移。过了 MMLU 的量化权重，可能在你用户真正会发的长尾推理题上翻车。把每个杠杆当成一次上线，而不是一次配置改动。下一章 [延迟工程](04-latency-engineering.md) 会在时间这条轴上遇到同一个模式。

## 参考文献

[^1]: Anthropic, "Prompt caching", official documentation, June 2026, https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
[^2]: OpenAI, "API pricing", June 2026, https://openai.com/api/pricing/
[^3]: Google, "Gemini API pricing", June 2026, https://ai.google.dev/gemini-api/docs/pricing
[^4]: Anthropic, "Message Batches API", official documentation, June 2026, https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
[^5]: Isaac Ong et al., "RouteLLM: Learning to Route LLMs from Preference Data", UC Berkeley Sky Computing Lab, July 2024, https://sky.cs.berkeley.edu/project/routellm/
[^6]: arXiv:2604.23577, "Closed-Loop LLM Routing with Conformal Cascading and Distillation Co-Optimization", April 2026, https://arxiv.org/abs/2604.23577
[^7]: Eldar Kurtic et al., "Give Me BF16 or Give Me Death? Accuracy-Performance Trade-offs in LLM Quantization", arXiv:2411.02355, November 2024, https://arxiv.org/abs/2411.02355
[^8]: Amazon Science, "Performance-Guided Knowledge Distillation for Efficient Text Classification", arXiv:2411.05045, November 2024, https://arxiv.org/abs/2411.05045
[^9]: Tian Pan, "Knowledge Distillation Economics", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-09-knowledge-distillation-economics-production-ai
