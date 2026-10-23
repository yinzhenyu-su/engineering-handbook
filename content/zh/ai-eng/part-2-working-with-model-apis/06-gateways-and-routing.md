---
title: "网关与路由"
description: "用一个 API 挡在所有供应商前面，先便宜后昂贵的级联，供应商故障切换，以及判断网关何时才值回运维成本的那条准则。"
slug: 06-gateways-and-routing
part: 2
chapter: 6
difficulty: intermediate
prerequisites:
  - 00-calling-models
  - 04-errors-retries-fallbacks
  - 05-choosing-models-and-reading-benchmarks
tags: [llm, http-api, api-gateway]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 网关与路由

周二下午，OpenAI 正赶上糟糕的一小时。它的状态页挂着黄色。你的应用通过官方 SDK 直接调用 `gpt-4o`，每个请求都在超时。同一个 vault 里放着 Anthropic 的密钥，你手上也有一个能用的 Bedrock 账号。但只要不发代码，这两个都用不上——因为你的调用点全都 `import openai`、把模型名写死在代码里，下游还按 OpenAI 的响应结构解析。等你开完分支、提了 PR、灰度发布完，故障早就结束了。

网关（gateway）存在的意义就是堵住这个故障模式。网关是一个与 OpenAI 兼容的单一端点，挡在所有你在意的供应商前面。你的应用像调用 OpenAI 一样调用它。底层它挑一个后端、必要时转换请求、把响应归一化，再把结果交给你。OpenAI 挂了，网关就把请求路由到 Anthropic。便宜模型够用时，它就用便宜模型。这些逻辑一行都不在你的应用代码里。

## 模式：一个端点，多个后端

机制是一个带三份职责的反向代理。前端它对上游说 OpenAI 的 `/v1/chat/completions` 协议，后端它对下游说每家供应商的原生 API，中间它持有一份路由（routing）策略，决定每个请求去哪儿。[^1] 你把现成的 OpenAI SDK 指向网关的 URL，换成网关的 API key，代码就不用再动了：

```python
from openai import OpenAI

# 同一个 SDK，同样的调用形状，只换 base_url。
client = OpenAI(
    base_url="https://your-gateway.example.com/v1",
    api_key="GATEWAY_KEY",
)

response = client.chat.completions.create(
    model="gpt-4o",  # 逻辑名；由网关决定真实后端
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
print("served by:", response.model)  # 究竟是哪个后端答的
```

响应里的 `model` 字段告诉你真正服务这个请求的是哪个后端。把它记进日志。少了这一行，你就没法调试一次悄悄换了供应商的故障回退（fallback），也没法审计哪个模型接触过哪段提示词。这是第一个该养成的习惯，要早于任何比“永远用 OpenAI”更花哨的路由规则。

网关有两种形态，它们的信任模型差别很大。**自托管**网关如 LiteLLM Proxy 跑在你自己的基础设施里，由一个 FastAPI 服务加一个 Postgres 数据库（做认证与花费统计）组成，可选再加 Redis 在多个副本间共享限流状态。[^1] 提示词永远不出你的 VPC。**托管**网关如 OpenRouter 或 Cloudflare AI Gateway 则以托管 SaaS 形式运行；零运维，但每段提示词都会流经第三方。[^2] OpenRouter 对充值收取 5.5% 手续费，推理定价按成本原样透传；Cloudflare 的核心功能免费，只有走 Unified Billing 时才收 5%。[^2][^3] 数据驻留是硬要求，或者你有运维余力并且想要零按请求加价，就选自托管。否则选托管——大多数团队都属于“否则”。

## 先便宜后昂贵：先试小模型

网关能给你的最有用的路由规则不是故障切换，而是省钱。应用流量的大多数都很简单：短查询、简单摘要、分类，小模型处理得绰绰有余。把这些全发给旗舰模型，等于用前沿的价格去买 Haiku 或 Llama-3-8B 就能答对的查询。这个模式叫级联（cascade）：先试便宜模型，只在便宜模型失败时才升级到贵的。

在 LiteLLM 里，这是两行配置：

```python
from litellm import Router

router = Router(
    model_list=[
        {
            "model_name": "cheap",
            "litellm_params": {
                "model": "groq/llama3-8b-8192",
                "api_key": "GROQ_KEY",
            },
        },
        {
            "model_name": "expensive",
            "litellm_params": {
                "model": "openai/gpt-4o",
                "api_key": "OPENAI_KEY",
            },
        },
    ],
    routing_strategy="cost-based-routing",
    fallbacks=[{"cheap": ["expensive"]}],
    num_retries=2,
    cooldown_time=30,
)
```

`cost-based-routing` 默认挑最便宜的健康部署。`fallbacks=[{"cheap": ["expensive"]}]` 的意思是：如果便宜模型把重试次数耗光，或者撞上不可重试的错误，就把请求提升到贵模型再跑一遍。[^1] 有两类错误值得单独分桶。提示词超出便宜模型的上下文窗口时触发 `context_window_fallbacks`；一家供应商的安全过滤拒掉了另一家会接受的内容时触发 `content_policy_fallbacks`。[^4] 设上 `enable_pre_call_checks: true`，路由器在能提前看出提示词装不下时就跳过便宜模型，省掉那一趟白跑的往返。

级联不是免费的。便宜模型没命中时，你要付的是两次模型调用的完整延迟，而不是一次，你的最坏情况延迟预算必须覆盖这条慢路径。粗略的经验法则：升级率维持在 20% 以下、两档价格差至少 5 倍时，级联才划算。升级率超过 20%，说明大多数请求都在白付便宜模型那一趟往返。价格差不到 5 倍，省下的钱盖不住延迟的代价。不管哪种情况，唯一能确定的办法是在自己的流量上测——这正是[错误、重试与故障回退](./04-errors-retries-fallbacks.md)打下的基础。

## 故障切换：三层，依次递进

故障切换是级联的防守型表亲。级联问的是“更便宜的模型能不能干这活”，故障切换问的是“这家供应商坏了，还有谁能答”。生产网关会按顺序跑三层故障切换，这三层你都得知道，因为它们的失效方式各不相同。

![一张纵向三层示意图，展示网关如何逐级升级一个失败的请求：第一层带退避重试同一个部署，第二层切换到同一模型在另一区域或另一供应商的兄弟部署，第三层把请求提升到完全不同的模型家族；每一层都标注了触发它的错误类型](../../../ai-eng/part-2-working-with-model-apis/images/06-failover-tiers.png)
*三层逐级上升的成本与风险：部署内重试、兄弟部署切换、跨模型提升。大多数故障在第一层或第二层就被接住了；第三层会改变答案。*

**第一层是部署内重试。** 这是针对瞬时故障的便宜重试层：429、5xx、网络抖动。LiteLLM 默认 `num_retries=3` 配指数退避，之后才放弃这个部署。[^1] 大多数供应商的小毛病都过不了这一层，用户根本不会察觉。

**第二层是兄弟部署切换。** 同一个模型有多个部署，挂在同一个逻辑名下面。你在 `litellm_params` 里用 `order: 1` 和 `order: 2` 声明它们。路由器会先把所有 `order=1` 的部署连同它们各自的重试全部耗尽，才提升到 `order=2`。[^1] 经典形态是 `gpt-4o` 同时跑在 OpenAI 和 Azure 上：同一个模型，不同的基础设施，彼此独立的故障模式。OpenAI 挂了，Azure 顶上。输出还是同一个模型，所以评测和审计链都完好。

```python
LITELLM_CONFIG = {
    "model_list": [
        {
            "model_name": "gpt-4o",
            "litellm_params": {
                "model": "openai/gpt-4o",
                "api_key": "OPENAI_KEY",
                "order": 1,           # 主用
            },
        },
        {
            "model_name": "gpt-4o",
            "litellm_params": {
                "model": "azure/gpt-4o",
                "api_base": "https://my-azure.openai.azure.com/",
                "api_key": "AZURE_KEY",
                "order": 2,           # 区域备份
            },
        },
        {
            "model_name": "gpt-4o-fallback",
            "litellm_params": {
                "model": "anthropic/claude-sonnet-4-5",
                "api_key": "ANTHROPIC_KEY",
            },
        },
    ],
    "router_settings": {
        "fallbacks": [{"gpt-4o": ["gpt-4o-fallback"]}],
        "cooldown_time": 5,
        "allowed_fails": 3,
    },
}
```

一个部署在一分钟内失败次数超过 `allowed_fails`，LiteLLM 就把它冷却 `cooldown_time` 秒（默认 3 次失败、5 秒）。[^1] 多副本部署时，把这份冷却状态通过 Redis 共享出去；否则每个副本都会各自重新发现这家供应商已死，你会有一小段时间被所有实例同时猛捶它。

**第三层是跨模型回退。** 这一层危险。上面那段 `fallbacks` 会在所有同模型选项都耗尽后，把失败的 `gpt-4o` 请求提升到 Claude。用户拿到了答案，网关也没倒，但模型身份在流中途变了。不同模型给出不同输出；一套假定 `gpt-4o` 输出的评测，现在实际在给 Claude 打分。写着“gpt-4o 作答”的审计日志是错的。这就是为什么记录 `response.model` 不容商量。对于模型身份涉及合规或评测的路径，干脆不放跨模型回退，让请求大声失败。尽力而为的路径可以保留第三层，要求可复现的路径不行。

## 网关何时值得这份复杂度

网关是真正的基础设施。自托管 LiteLLM 意味着一个 FastAPI 服务加 PostgreSQL 加可选的 Redis，全都要部署、扩容、监控，并且每次供应商发一个破坏性变更就得升级。托管网关省掉了运维，但给你的提示词路径加了一个第三方，还加了一笔按请求的加价。两个选项都是税。问题是这笔税值不值。

诚实的准则是：下面三条里至少满足两条，网关才值回成本。[^5]

- **你调用了不止一家供应商。** 一家供应商只需要一层包装。两家就是分叉问题；每新增一家，定制重试、密钥和 schema 代码的面积就翻一倍。
- **供应商故障会伤到业务。** 如果 OpenAI 挂了你会凌晨 3 点被人叫起来，你就需要故障切换。定制的故障切换代码会漂移，网关的故障切换是配置。
- **你没法在 30 秒内答出“每个客户花了多少钱”。** 按请求归集成本是每个网关都免费提供的功能，而几乎没有团队自己写得好。

只调用一家供应商、还在做原型、或者有第三方网关满足不了的数据驻留规定，那就彻底跳过网关。“你不需要的基础设施就是税”这条规则在这个阶段格外管用。[^5]

延迟这笔账是让人安心的。LiteLLM 公布的基准测试在四台 4 核实例、1170 RPS 下，代理开销中位数 2 ms，p95 为 8 ms，p99 为 13 ms。[^6] Portkey 文档给出的额外延迟是 20-40 ms。[^7] 跟供应商 API 调用那几百毫秒到好几秒的长尾相比，网关就在噪声里。真正伤性能的不是网关，是选错了路由策略。LiteLLM 的文档明确把 `usage-based-routing` 标为“bad for perf”，因为它给每个请求都加上 Redis 往返。[^1] 热路径用 `simple-shuffle` 或 `latency-based-routing`；把按用量路由留给合同上确实要求 RPM/TPM 公平性的罕见场景。

> [!WARNING]
> **静默替换模型是唯一会搞坏评测的网关失效模式。** 本该发给 `gpt-4o` 的请求，因为回退链被触发，由 Claude 答了。评测集看到的是一堆不同模型的输出混在一起，你的分数开始漂移，而没人能追出原因。永远从 `response.model`（OpenRouter）或 `x-litellm-model-id` 响应头（LiteLLM）记下实际服务的模型，一旦与请求的不符就告警。对于模型身份属于契约一部分的路径，比如受监管的审计或模型锁定的评测，就在那条路径上关掉跨模型回退，哪怕这意味着故障期间直接返回错误。

你采用的第一个网关应该贴合你现有的工作方式。你在用 Cloudflare，AI Gateway 核心层免费，离你现有的边缘只有一跳 DNS。[^3] 你想要广泛的供应商覆盖和完全的控制权，LiteLLM Proxy 有最长的供应商名单（100+）和最活跃的开源社区。[^1] 你需要 SOC 2、ISO 27001、SSO 和审计日志又不想自己运维，Portkey 或 OpenRouter 能覆盖这块。[^2][^7] 你不必一次选对；这个模式的全部意义就在于换网关是改配置，不是重写。要避免的错误不是选错网关，而是自己写一套重试-回退-密钥轮换-成本追踪的层，管它叫“很薄”，然后眼看着它两个季度后长成这些工具中某一个的更差版本。

## 参考文献

[^1]: BerriAI, "LiteLLM Proxy: Load Balancing and Routing", docs.litellm.ai, https://docs.litellm.ai/docs/routing (fetched June 2026)
[^2]: OpenRouter, "Frequently Asked Questions", openrouter.ai, https://openrouter.ai/docs/faq (fetched June 2026)
[^3]: Cloudflare, "AI Gateway Pricing", developers.cloudflare.com, https://developers.cloudflare.com/ai-gateway/reference/pricing/ (last updated May 19, 2026)
[^4]: BerriAI, "LiteLLM Fallbacks", docs.litellm.ai, https://docs.litellm.ai/docs/proxy/reliability (fetched June 2026)
[^5]: LLM Gateway, "LLM Gateway vs Direct API: When the Provider SDK Stops Scaling", llmgateway.io, https://llmgateway.io/blog/llm-gateway-vs-direct-api (April 2026)
[^6]: BerriAI, "LiteLLM Benchmarks", docs.litellm.ai, https://docs.litellm.ai/docs/benchmarks (fetched June 2026)
[^7]: Portkey AI, "What is Portkey?", portkey.ai, https://portkey.ai/docs/introduction/what-is-portkey (last modified January 28, 2026)

