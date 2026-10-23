---
title: "成本建模与定价"
description: "为什么边际成本重新变得重要。按任务（而不是按 token）算成本、四种定价架构，以及判断一个功能是否贵到不该上线的算术。"
slug: 04-cost-modeling-and-pricing
part: 11
chapter: 4
difficulty: intermediate
prerequisites:
  - part-1-how-models-behave/05-the-cost-quality-latency-triangle
  - part-2-working-with-model-apis/07-prompt-caching
  - part-2-working-with-model-apis/06-gateways-and-routing
tags: [llm, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 成本建模与定价

2025 年 6 月，一位 Cursor 用户收到一张 $7,225 的账单，只来自一个月的 agent 循环。[^1] 同一位用户在上一版定价模型下——高级模型请求每次固定收 $0.04——一直在跑大上下文的 agent 循环，实际上等于 Cursor 在补贴他。公司改成与 API 等价的直通费率后，用户实际拿到的用量没变，用户*为之付费*的账变了。

这就是 AI 产品单位经济（unit economics）的新形态。每次请求的边际成本（marginal cost）不会随着规模扩大被摊薄到接近零。每一次查询都把真实的 token 发给提供商，产生真实费用，而且这笔费用几乎随上下文长度和输出长度线性增长。传统 SaaS 的毛利率（gross margin）在 70-90%；以 AI 为主的公司落在 50-60%，这个差距几乎全部来自推理（inference）开销。[^2]

本章讲三个决策：用业务能据以行动的度量方式衡量成本；选一种不会悄悄把毛利漏给重度用户的定价模型；以及为一个功能在你想要的价位上是否贵到不该上线，定出一个数字门槛。缓存的深层机制见[提示词缓存](../part-2-working-with-model-apis/07-prompt-caching.md)，路由的实现见[网关与路由](../part-2-working-with-model-apis/06-gateways-and-routing.md)。这里要做的是压在它们之上的那层算术。

## 按 token 算成本，单位就错了

提供商按 token 计费，开发者交付的是任务。两者之间的换算，正是大多数成本分析崩掉的地方。

天真的公式看起来人畜无害：

```text
cost_per_call = (input_tokens * input_price + output_tokens * output_price) / 1e6
```

但一个任务不是一次调用。它是每一轮都重发的系统提示词，是不断累加的对话历史，是工具 schema（Anthropic 按模型和选项不同，每次调用把工具 schema 计为 290-497 个输入 token 的额外开销[^3]），是检查输出质量时的那次校验调用，以及校验失败后的重试。正确的单位是经过可靠性修正的“每个成功任务成本”：

```text
cost_per_task = sum(cost_per_call for all calls in the workflow) / p_success
```

p_success = 0.85 的工作流，真实每任务成本比天真的单次调用估算高出 18%。Zartis 记录过一个案例：每天 $200 的推理 token，换来每天 $20,000 的人工补救成本（每天 10,000 个任务，每次失败花 5 分钟，人力全成本 $80/小时）。API 看板上只显示那 $200，那 $20,000 散落在 Jira 工单里。[^4]

多轮工作流里另一个无声的漏点是上下文累积。一场 10 轮、每轮 200 token 的对话，到第 10 轮要重发 2,000 token 的历史，第 9 轮 1,800 token，以此类推。除了对话记录里天真可见的部分，还多出 11,000 个隐藏输入 token。按 Sonnet 4.6 的输入价格（2026 年 6 月为 $3/MTok[^3]），除非你对每次调用都做埋点，否则每场 10 轮对话里就有 $0.033 看不见的额外开销。

```python
def cost_per_task(system_prompt_tokens, user_tokens, output_tokens,
                  input_price_per_mtok, output_price_per_mtok,
                  cache_hit_rate=0.0, p_success=1.0):
    cache_read_price = input_price_per_mtok * 0.10  # Anthropic 缓存读取按 10% 计价
    cached = system_prompt_tokens * cache_hit_rate
    uncached = system_prompt_tokens * (1 - cache_hit_rate) + user_tokens
    input_cost = (uncached * input_price_per_mtok + cached * cache_read_price) / 1e6
    output_cost = output_tokens * output_price_per_mtok / 1e6
    return (input_cost + output_cost) / p_success

# Haiku 4.5 上的客服分类器：$1/$5 每 MTok，2026 年 6 月
print(f"${cost_per_task(500, 150, 80, 1.0, 5.0, cache_hit_rate=0.8, p_success=0.95):.6f}")
```

## 用 2026 年 6 月的价格实算一遍

三个参照点撑起本章余下的内容。所有价格均为 2026 年 6 月的标准档。[^3][^5]

**Haiku 4.5 上的客服分类器**（$1/$5 每 MTok，500 token 系统提示词的缓存命中率 80%）。每次调用成本 $0.000690。p_success = 0.95 时，每任务 $0.000726。每天 50,000 个任务，就是每天 $36，约每月 $1,100。

**GPT-5.4 上的文档摘要**（$2.50/$15 每 MTok，每份文档都不同，所以不用缓存）。5,000 个输入 token，400 个输出 token。每次调用 $0.019。p_success = 0.90 时，每任务 $0.021。每天 1,000 个任务，每月 $640。

**Claude Fable 5 上带工具的多轮 agent**（$10/$50 每 MTok，5 轮，每轮平均 2,000 个上下文 token 加 300 个用户 token 加 497 个 token 的工具开销，每轮输出 500 个 token）。每轮成本 $0.053。五轮 $0.265。p_success = 0.80 时，每任务 $0.331。这正是前沿模型的 agent 任务在标准 SaaS 毛利下，面对任何低于约 $300/月 ARPU 的产品都变得经济上勉强的区间。

批量 API（batch API）折扣三家提供商都有：异步、最长 24 小时返回的负载，标准价五折。[^3][^5][^6] 示例 B 的文档摘要走批量后，从每任务 $0.021 降到约 $0.011。24 小时的 SLA 让批量不适合交互式功能，但对夜间报表、批量分类和离线评测正好解锁。

## 交叉补贴问题

每个用户的任务量不是正态分布。前 5-10% 的用户通常消耗掉 50-80% 的 token 支出。[^2] 一个按中位用户设计的固定月费产品，能在中位数上显示正毛利，同时在前十分位上亏钱。

```python
def margin_per_user(arpu, tasks_per_month, cost_per_task, infra_overhead=0.05):
    llm_cost = tasks_per_month * cost_per_task
    infra_cost = arpu * infra_overhead
    cogs = llm_cost + infra_cost
    margin = (arpu - cogs) / arpu if arpu > 0 else 0
    return {
        "llm_cogs": llm_cost,
        "gross_profit": arpu - cogs,
        "gross_margin_pct": round(margin * 100, 1),
    }

# 每月 $20 套餐，中位用户 50 个任务，每任务 $0.005：很舒服
print(margin_per_user(20.0, 50, 0.005))
# 同一套餐，P95 用户 500 个任务：仍为正，但很薄
print(margin_per_user(20.0, 500, 0.005))
# 重度用户 5,000 个任务：深度为负
print(margin_per_user(20.0, 5000, 0.005))
```

Requesty 对 1,000 多名使用 AI 编码 agent 的开发者所做的分析（2026 年 5 月）让这个分布落了地：每个活跃用户的 LLM 平均成本是 $92/月，Claude Code 用户升到 $108/月，P95 用户达到 $291/月。[^1] 一个每席位每月收 $20、却要为一个 P95 用户花 $291 API 费用的产品，无论当初基于什么固定月费假设，这些账户上的毛利都是深度为负。

## 四种定价架构

定价模型的选择与此前任何 SaaS 定价决策在结构上都不同：提供商的成本随用户行为连续变化，而固定席位费带来的是固定收入。生产环境里的做法可以归为四种架构。

**按席位（打包固定月费）。** 所有 token 成本由厂商吸收。最好卖、上手最快、计费零摩擦。失败模式是隐形的毛利压缩：当某一批用户用量飙升时，毛利静悄悄地恶化。$50/用户/月、却被吃掉 $30 LLM 费用的套餐是有记录的案例。[^2] 按席位可行的前提是你能通过产品设计严格约束用量（固定输出长度、窄范围），而且 LLM 成本低于订阅收入的 10-15%。

**按用量 / 计量超额。** 基础订阅覆盖一定的 token 或任务额度，超出部分按单位付费。提供商的成本结构直接映射到用户账单。失败模式是账单惊喜：不盯用量的用户收到意外的大额账单，然后流失或投诉。Cursor 2025 年 6 月的定价调整是典型案例：从每次请求固定 $0.04 改成与 API 等价的直通计费，引发强烈反弹，包括本章开头那张 $7,225 的账单。[^1] 计量计费可行的条件是：计价单位对用户可读（token、任务、调用次数），用户对用量有足够控制权、使计量账单显得公平，产品还提供实时花费可见性。GitHub Copilot 在 2026 年 6 月把所有套餐改成按用量的 credit（额度），反映出行业对这类“重度用户会带来不成比例成本”的工具正朝这个模型收敛。

**按 credit 计费。** 用户预先买一池 credit，AI 调用从中扣减。credit 把面向用户的价格单位与 token 解耦。失败模式是不透明：用户不明白为什么一段更长的对话要扣更多 credit，而厂商的成本结构藏在一个任意的换算系数后面。把 credit 按 token 映射，等于把成本风险转给客户，也压低付费意愿。[^7] 有效的做法是按价值映射：1 credit = 解决一张客服工单、通过一份文档、发布一份草稿。Lovable 按映射到“有意义的构建”的 credit 收费，而不是按 token。[^8]

**按结果计费。** 只在某个定义好的下游结果发生时收费。Intercom 的 Fin 对每通完全解决（无需人工介入）的对话收 $0.99，每月前 50 次解决免费，另加 $49/月的基础费。[^9] 厂商承担所有失败尝试的 LLM 成本。按结果计费可行的条件：平台已经在追踪一个可观测的二值结果；厂商能可靠地建模失败率，从而把结果价格定在盈亏平衡之上；客户 ROI 明确与结果数量挂钩。Intercom 在这条路走通之前，花了好几年搭结果追踪的基础设施。

截至 2026 年，主流做法是混合制。56% 的 AI SaaS 公司在基础订阅里给出明确用量额度，超出后按计量超额或 credit 钱包计费。[^2] 对多数 ARPU 在 $20-$99/月的 AI 重度产品，混合制是正确的默认值，因为它是唯一既不必把功能限死、也不必指望重度用户恰好停在边缘的架构。

## 什么时候一个功能贵到不该上线

决策规则形式很干净。当某功能在 P90 用户行为下的每任务成本，不会把毛利率压到产品底线之下时，它就可以上线。底线的位置取决于公司画像：以 SaaS 为主的公司 60-70%，以 AI 为主的公司 40-50%。[^2]

```python
def ship_decision(arpu, tasks_per_month, cost_per_task,
                  target_margin=0.60, infra_overhead=0.05):
    max_total_cogs = arpu * (1 - target_margin)
    max_llm_budget = max_total_cogs - arpu * infra_overhead
    max_cost_per_task = max_llm_budget / tasks_per_month if tasks_per_month > 0 else 0
    actual_cogs = cost_per_task * tasks_per_month + arpu * infra_overhead
    actual_margin = (arpu - actual_cogs) / arpu
    return {
        "max_cost_per_task_for_target_margin": round(max_cost_per_task, 6),
        "actual_margin_pct": round(actual_margin * 100, 1),
        "verdict": "SHIP" if actual_margin >= target_margin else "HOLD",
    }
```

算下来是 HOLD 时，按代价从小到大依次用这些手段：

1. **降级模型。** 在质量差异用户看不出来的任务上，把前沿模型换成更小的模型。Haiku 4.5 的输入和输出价格都只有 Sonnet 4.6 的 1/5。[^3]
2. **提示词缓存。** Anthropic 的缓存读取按标准输入价的 10% 计费；缓存写入按 125%（5 分钟 TTL）或 200%（1 小时 TTL）计费。在 Haiku 4.5 上，500 token 的系统提示词做到 80% 命中率，可把有效输入成本从 $1/MTok 压到 $0.28/MTok。[^3]
3. **上下文压缩。** 把较早的对话轮次做摘要，而不是重发完整记录。这砍掉了多轮工作流上的上下文税。
4. **模型路由。** 简单请求走便宜模型，复杂请求走前沿模型。一个把 70% 流量路由到便宜 5 倍档位的复杂度分类器，在质量守得住的前提下能把整体 LLM 销货成本（COGS）砍掉约 60%。
5. **批量 API。** 非交互式负载走五折的批量档。
6. **限流与用量上限。** GitHub Copilot 免费版每月 2,000 次补全加 50 条对话消息，是明确的经济约束，不是产品决策。[^2]
7. **重新定价。** 涨价，或把用户迁到更高档位。Intercom 的定价重置（涉及约 1,500 万美元 ARR）说明，只要做得透明，重新定价是可行的。[^10]
8. **不上线。** 一个在 $99/月能赚钱、在 $20/月就亏钱的功能，是定价决策的问题，不是能力边界的问题。

Intercom 内部被引用最多的一个数据：早期 co-pilot 里的 AI 建议提示词消耗了近 50% 的运营成本，而用户采用率不到 1%。砍掉这一个功能让产品毛利翻倍，用户侧毫无感知。[^10]

“补贴式 AGI”（subsidized AGI）这个失败模式值得点名，因为它会把团队引向错误结论。截至 2026 年中，ChatGPT Pro 每月 $200，满负荷使用时毛利率是 -1,650%；Claude Max 20x 每月 $100，毛利率是 -900%。[^11] 只要单个重度用户超过宣传容量的 10-20%，每一档消费级 AI 订阅都会跌进负毛利。这些公司是在有意补贴用量以推动采用。没有那张资产负债表的创业公司，除非为这笔补贴备好了明确的资金，否则不该照搬这个模型。做任务级经济测算时，用提供商的 API 价格，不要用消费级订阅价格。

## 参考文献

[^1]: Requesty, "The Coding Agent Economy", May 2026. https://www.requesty.ai/coding-agent-economy

[^2]: Tian Pan, "Pricing Your AI Product: Escaping the Compute Cost Trap", April 16, 2026. https://tianpan.co/blog/2026-04-16-pricing-ai-product-compute-cost-trap

[^3]: Anthropic, "Pricing", API Documentation, June 2026. https://docs.anthropic.com/en/docs/about-claude/pricing

[^4]: Zartis, "AI Agent Cost Optimisation: Why Token Cost Is the Wrong Number to Optimise", 2026. https://www.zartis.com/ai-agent-cost-optimisation-why-token-cost-is-the-wrong-number-to-optimise/

[^5]: OpenAI, "API Pricing", June 2026. https://openai.com/api/pricing/

[^6]: Google, "Gemini Developer API Pricing", June 2026. https://ai.google.dev/gemini-api/docs/pricing

[^7]: Apptension, "AI Unit Economics: Measure Cost Per Task, Not Tokens", 2026. https://apptension.com/guides/ai-unit-economics-cost-per-task-not-tokens

[^8]: Freemius, "AI App Pricing Patterns", 2026 (Lovable credit-mapping example).

[^9]: Intercom, "Fin AI Agent Pricing", June 2026. https://www.intercom.com/fin

[^10]: Chargebee, "How Intercom Built Its Outcome-Based Pricing Model for AI", December 2025. https://www.chargebee.com/blog/how-intercom-built-its-outcome-based-pricing-model-for-ai/

[^11]: Business Engineer, "The Subsidized AGI Economy", mid-2026. https://businessengineer.ai/p/the-subsidized-agi-economy
