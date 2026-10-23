---
title: "托管还是自托管"
description: "决定自托管何时胜过按 token 计费的四行式 TCO 模型、为什么 GPU 利用率是无声的乘数，以及 2026 年的默认做法：先托管，再迁移热路径。"
slug: 00-hosted-vs-self-hosted
part: 10
chapter: 0
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/06-gateways-and-routing
  - part-2-working-with-model-apis/07-prompt-caching
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 托管还是自托管

2026 年初，一家 FinTech 团队用 GPT-4o 搭了个 AI 聊天机器人。第一个月 $15K，第三个月 $60K，按年推算 $700K。他们慌了，租了八块 H100，把所有东西都自托管。六个月后，每月要花 $85K，天天跟 CUDA 崩溃搏斗，还有一次事故让产品停了九个小时。[^1]

解法不是托管，也不是迁移，而是两个都做。他们把每天 60 万次批量分类调用路由到自托管的 13B 模型上，把每天 5,000 次复杂查询留在前沿 API 上。总支出降到 $8K。第一次的错误不是自托管，而是把*所有东西*都自托管。

这一章讲的就是那家 FinTech 团队错了两次才做对的那道决策。它也是你作为 AI 工程师要做的最有分量的一个基础设施选择，而且是最多团队两个方向都会搞砸的那一个。

## 两行式 TCO 陷阱

陷阱在于拿 GPU 租金去跟 API 的 token 支出对比，然后得出自托管更划算的结论。算式长这样：

```text
8x H100 on-demand:    $25,200/month
Equivalent API spend: $30,000/month
Savings:              $4,800/month, right?
```

这个两行模型是错的，因为它漏掉了账单上最大的一项：跑集群的那些人。正确的总拥有成本（TCO）有四行。[^2]

| 项目 | 占自托管 TCO 的比例 | 实际开销 |
|---|---|---|
| GPU 租金 | 60-70% | 8 块 H100 按需实例放在新兴云（neocloud）上，每月 $25K（2026 年 4 月） |
| 推理工程师 | 25-30% | 美国资深工程师，含各项成本每月 $20-30K |
| 服务栈运维 | 5-10% | 每周 8-12 个工程师小时，用在调 vLLM、容量和可观测性上 |
| 自建的机会成本 | 视情况 | 两个月没能发产品 |

打破那套天真算式的是工程师这一行。一个每月在闭源 API 上花 $8K 的团队，没法为把它搬回自建而论证一个每月 $25K 的工程师：光工程师的价钱就是所省下金额的三倍。API 支出低于大约每月 $50K 时，四行模型给的结论是：别迁移，去路由。[^1]

由此得出的决策规则：

- **API 支出约每月 $10K 以下。** 留在 API 上。把工程师成本算进去，自托管要贵 12-30 倍。这题连算都不用算。
- **每月 $10K 到 $50K。** 把大批重复任务路由到自托管或托管的开放权重模型上；复杂任务留在前沿 API 上。在这个档位，混合路由通常能把支出砍掉 40-70%。
- **每月 $50K 以上。** 大批流量自托管是保住毛利的必需项。跟省下来的钱比，工程师成本不值一提。

这是用量这一维。还有另外两维。

## 利用率是无声的乘数

一块 GPU 在 10% 负载和 100% 负载下按同样的时价计费。就这一条事实，决定的自托管成败比任何基准测试都多。

![一张柱状图，展示五种 GPU 利用率水平下的有效每千 token 美元成本，一条水平虚线标出等效的高端 API 价格；10% 那根柱子高耸在虚线之上，60% 到 100% 的柱子则稳稳落在虚线之下。](../../../ai-eng/part-10-deployment-and-cost/images/00-utilization-multiplier.png)
*有效每千 token 成本与利用率成反比。持续 10% 负载下，一块 H100 的每输出 token 成本比高端 API 还贵。*

算式很直白。一块跑 Llama 3.1 70B 的 H100，峰值大约每秒 1,400 个 token。[^3] 在新兴云上每小时 $2.90、利用率 100%，折合每 1K token 约 $0.0006。利用率掉到 10%，每 1K 的成本就升到 $0.006。而 DeepSeek V4 Flash 每百万输入 token 才 $0.14——你空转的 H100 突然比托管 API 还贵。

生产里的 LLM 负载是突发式的。把流量波动和 SLA 驱动的预留都算进去，多数团队的平均利用率落在 40-65%。[^4] 按突发负载下的 P99 长尾延迟来定容量，平均利用率只能到 35-45%，因为相比按 P50 定容量，你需要多留 30-40% 的余量。[^2] “我们会把集群跑到 80%”这种计划，碰到真实流量很少有活下来的。

托管 API 的供应商没有这个问题。他们把一块物理 GPU 摊到几千个租户身上，一路跑满。正是这个结构性优势，让廉价的开放权重 API（Together AI、Fireworks、DeepSeek）在多数现实用量下持续压低单租户自托管的价格。

决策规则：在承诺 GPU 容量之前，先量三个月的生产实际 token 用量。如果预测利用率低于 60%，托管更便宜。别拿六周的预测去订一年的集群。

## 合规比成本更早做决定

有些负载的选择已经被替你做完了。如果你处理美国受保护健康信息（PHI）却没签业务伙伴协议（BAA），那么在跑任何成本分析之前，你已经在违反 HIPAA。如果你把欧盟个人数据传输到美国的基础设施上，却没有 2021 版的《数据处理协议》（DPA）和标准合同条款（SCC），那就是违反 GDPR。这两件事都不需要真出事才产生责任。

OpenAI、Anthropic、Google 的标准消费级 API 默认不提供 BAA。企业级有时提供：带 BAA 的 AWS Bedrock、带 BAA 的 Azure OpenAI、带 BAA 的 Google Vertex AI。在接任何管道之前，先把协议落到书面。有“SOC 2 认证”不等于“愿意签 BAA”。

对于有严格驻留要求的欧盟个人数据，可选模型清单会缩到云厂商在欧洲区域实际跑的那些，而最新模型比美国晚上几周到几个月。欧盟《人工智能法案》又加了一层：针对高风险系统的第 10 条数据治理要求自 2026 年 8 月 2 日起可强制执行，罚则最高 1500 万欧元或全球营业额的 3%。

顺序很重要。先跑合规检查。

```python
def compliance_gate(
    has_phi: bool,
    baa_available: bool,
    has_eu_personal_data: bool,
    dpa_and_scc_available: bool,
    strict_eu_residency: bool,
) -> dict:
    """Returns whether compliance forces self-hosting or VPC deployment.
    Run this BEFORE any TCO calculation.
    """
    blockers = []
    if has_phi and not baa_available:
        blockers.append("HIPAA: PHI with no BAA. Use enterprise tier or self-host in VPC.")
    if has_eu_personal_data and not dpa_and_scc_available:
        blockers.append("GDPR: EU personal data with no DPA+SCC. Cross-border violation.")
    if strict_eu_residency:
        blockers.append("EU residency: inference must run in EU region or self-host in EU.")
    return {"forced_private": bool(blockers), "blockers": blockers}
```

如果这道闸门亮了，你不需要盈亏平衡分析。你需要的是 VPC 部署，或者一个愿意签对文书的企业级供应商。只有当合规的答案是“没有约束”时，成本算式才需要算。

## 先托管，再迁移热路径

对其他人来说，2026 年的默认做法一点也不浪漫：第一版整个建在托管 API 上。给每一次调用打点。在生产里跑上两三个月，再去看分布。

你会发现每个成熟团队都会发现的那件事：大约 80% 的 token 来自 20% 的查询类型。[^1] 那些高频、低方差的查询（分类、抽取、摘要、RAG 排序）就是自托管的候选。另外那 20% 复杂、敏感或量少的查询，应该留在前沿 API 上。这不是妥协，这是正确的架构。

混合栈活在路由器里。它按任务类型、数据敏感度、预期难度和失败代价给每个请求分类，然后挑一个模型。[^5] 升级条件要具体，不能靠感觉：测试生成走本地，但测试连着两次不过或者覆盖率没达标就升级到前沿模型。批量抽取走本地，但 schema 校验失败就升级。盯的是每个被接受结果的平均成本，不是每 token 成本；只有当本地模型的重试没把省下的钱吃掉时，它才真的更便宜。

下面这些才是真该扣下迁移扳机的时候：

- **每月 API 支出超过 $10K**，而且大头是大批重复任务
- **团队里已经坐着一位推理工程师**，之前在生产里跑过 vLLM 或 SGLang 集群（不是“我们打算招一个”，那是三到六个月的爬坡）
- **三个月的实测用量**显示持续负载放到目标集群规格上能超过 60% 的 GPU 利用率
- **合规允许**（先跑那道闸门）

以下情况别扣扳机：

- 用量还在飞快增长，或者产品正在转向（迭代期里，模型升级跑步机的成本太高）
- 全部用量都是量少的复杂任务（没有大批量可以路由给便宜模型）
- 团队既没有推理基础设施经验，也没有招人的预算

## 一个值得认真对待的反调

前沿的托管模型不只是权重。它还带着 API 一并提供的东西：安全后训练、工具使用调优、上下文管理、多模态处理，以及持续更新。8 块 H200 跑 Qwen3-235B，在多数任务上是“差不多 Sonnet”的水平，但在最难的推理上仍然替代不了 GPT-5.5。[^5] 如果你的产品在每一个查询上都依赖前沿能力，自托管的总成本可能反而更高：本地模型需要更多重试和人工复核，每个成功结果的成本会超过表面上省下的每 token 差价。

对应的规则：质量底线用你自己产品的评测来定，不用替代性基准。如果本地模型通过了你路由给它的那类任务的评测集，那就是底线。没过就升级。别因为每 token 的账好看，就让一个 13B 模型去干前沿模型干的活。

## 部署底座问题

这些东西到底跑在哪？老实的回答是“容器，由 Kubernetes 编排，跑在 GPU 节点池上”，而这一句就是本书在部署底座（deployment substrate）上的全部范围。Pod 调度、自动扩缩、GPU 节点管理、入口、服务网格：那些属于通用基础设施工程，不属于 AI 工程。放到架构尺度上，[AI 负载的部署底座](../../hld/part-9-ai-ml-system-design/00-llm-serving-architecture.md)讲的是白板视角。

你确实需要了解的 AI 基础设施专门知识（生产环境的向量存储、语义缓存、agent 任务队列、embedding 管道）在[AI 基础设施模式](07-ai-infrastructure-patterns.md)里。服务栈本身（vLLM、KV cache、连续批处理）是下一章。

那家 FinTech 团队花大钱换来的一点认识：这不是二选一，这是一个路由器。第一版建在托管 API 上，埋好点，找出热路径，然后迁移它。先跑合规闸门，再跑成本算式。容量按长尾延迟来定，别按你希望自己拥有的那个利用率数字来定。

## 参考文献

[^1]: Tian Pan, "The Build-vs-Buy LLM Infrastructure Decision Most Teams Get Wrong", TianPan.co, April 15, 2026, https://tianpan.co/blog/2026-04-15-build-vs-buy-llm-infrastructure
[^2]: Digital Applied, "Self-Hosting Frontier AI Models: 2026 TCO Analysis", April 24, 2026, https://www.digitalapplied.com/blog/self-host-frontier-models-tco-analysis-2026
[^3]: Digital Applied, "Self-Hosting Open-Weight LLMs: When It Actually Wins", May 27, 2026, https://www.digitalapplied.com/blog/self-hosting-open-weight-llms-2026-deployment-decision-guide
[^4]: Spheron, "LLM Inference On-Premise vs GPU Cloud: 2026 Cost and Break-Even Analysis", April 13, 2026, https://www.spheron.network/blog/llm-inference-on-premise-vs-cloud/
[^5]: TensorOps, "Is It Time to Self-Host LLMs? An Enterprise Guide", May 31, 2026, https://tensorops.ai/blog/is-it-time-to-self-host-llms
