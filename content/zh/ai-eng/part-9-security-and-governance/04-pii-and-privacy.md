---
title: "PII 与隐私"
description: "在调用模型之前完成检测与脱敏，理清控制者—处理者—次级处理者这条链，并选对数据保留与数据驻留的开关。"
slug: 04-pii-and-privacy
part: 9
chapter: 4
difficulty: advanced
prerequisites:
  - part-2-working-with-model-apis/00-calling-models
  - part-2-working-with-model-apis/07-prompt-caching
  - 03-abuse-prevention
tags: [llm, ai-safety]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# PII 与隐私

用户把一条客户记录粘进了你的客服机器人。这个字符串里现在有客户姓名、邮箱、电话，可能还有一个带账单明细的订单 ID。它从浏览器传到你的应用服务器，再从那里传到第三方模型 API，然后继续传到那家服务商的云基础设施，可能还会落到另一个国家的内容审核外包商手里。每经过一跳，数据都可能被写进日志、被保留一段时间、被路由跨过国境，或者被人短暂地人工看一眼。你的 DPO 需要你让这整条链路同时满足 GDPR 第 28 条、CCPA，以及（如果数据涉及健康信息）HIPAA。

工程上的决策很具体。在模型看到数据之前就检测出 PII（个人身份信息）。按模型是否需要真实值，选择脱敏（redact）、假名化（pseudonymize）或直接透传。在服务商那边配好数据保留与驻留。核验 DPA 之下深达四层的合同链条。这些单独做任何一项都不完整；合在一起，才是针对那个唯一真会出现在监管函里的漏洞的纵深防御（defense in depth）。

## 先检测，再决策

Microsoft Presidio 是 Python 生态里做 PII 检测的开源默认选择（Apache 2.0，适合以 sidecar 方式部署，开发和生产跑的是同一套）。两个引擎：`AnalyzerEngine` 找出片段并打上标签，`AnonymizerEngine` 负责变换它们。[^1]

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

def redact_before_call(text: str, language: str = "en") -> tuple[str, list]:
    results = analyzer.analyze(text=text, language=language)
    redacted = anonymizer.anonymize(text=text, analyzer_results=results)
    return redacted.text, results

raw = "Hi, my SSN is 078-05-1120 and email is alice@example.com"
clean, _ = redact_before_call(raw)
# clean -> "Hi, my SSN is <US_SSN> and email is <EMAIL_ADDRESS>"
```

`AnalyzerEngine` 先把输入过一遍 NLP 后端（默认是 spaCy 的 `en_core_web_lg`），再交给一组识别器（recognizer）。有些是带校验和的模式匹配（`CREDIT_CARD` 走 Luhn 校验，`IBAN_CODE` 走 mod-97 校验，`US_SSN` 校验区段号）。有些基于 NER（`PERSON`、`LOCATION`）。`ContextAwareEnhancer` 会按邻近词加权（9 位数字前面出现“SSN”，置信度就上调）。[^2] 开箱即用时，Presidio 覆盖 13 种全球实体，另外还有针对美国、英国、西班牙、意大利、波兰、新加坡、澳大利亚、印度、韩国、尼日利亚、泰国等司法辖区的识别器。

默认置信度阈值是 0.35，取值区间 0 到 1。在 HIPAA 或金融场景下调到 0.30 左右，用一部分误报换更高的召回率。普通聊天场景保持默认值，过度脱敏会伤体验。Presidio 自己的文档对局限说得很直白：“there is no guarantee that Presidio will find all sensitive information.”[^1] 高风险部署里要再叠一层 Azure AI Language PII 或领域专用模型（临床文本的默认选择是 HuggingFace 上的 `blaze999/Medical-NER`）。

运维提示：在进程启动时只实例化一次 `AnalyzerEngine()`。首次调用要加载 spaCy 的 `en_core_web_lg`，大约 500 ms、1 GB 内存。如果有多进程，就把 Presidio 跑成 sidecar HTTP 服务（`docker run -p 5002:3000 presidio-analyzer`）。

## 脱敏、假名化，还是原样透传

生产里会用到三种模式。分界线是：模型要完成任务，是否必须拿到原始值。

**脱敏（redact）** 不可逆。把 `Alice Smith` 换成 `<PERSON>`。模型永远看不到真名。这适用于 FAQ 机器人、通用摘要器，以及回答客户记录相关政策问题的合规助手——它们不需要知道是谁的记录。一旦模型需要用名字称呼用户，或者要写出会被拼回原记录的文本，这招立刻失效。

**假名化（pseudonymize）** 用可逆的 token 替换。把 `Alice Smith` 换成 `<PERSON_1>`，映射存进会话级的 dict，把假名化后的文本发给模型，等响应回来再把原文替换回去。模型可以在一次对话里一致地称呼 `<PERSON_1>`；服务商始终看不到真名。凡是需要指代连贯的对话型应用，这都是默认做法。

```python
from presidio_anonymizer.entities import OperatorConfig

_mapping: dict[str, str] = {}
_counter: dict[str, int] = {}

def pseudonymize(text: str, language: str = "en") -> tuple[str, dict]:
    results = analyzer.analyze(text=text, language=language)

    def make_token(entity_type: str, original: str) -> str:
        key = f"{entity_type}:{original}"
        if key not in _mapping:
            _counter[entity_type] = _counter.get(entity_type, 0) + 1
            _mapping[key] = f"<{entity_type}_{_counter[entity_type]}>"
        return _mapping[key]

    operators = {
        "DEFAULT": OperatorConfig(
            "custom",
            {"lambda": lambda x: make_token("ENTITY", x)},
        ),
    }
    out = anonymizer.anonymize(text=text, analyzer_results=results, operators=operators)
    return out.text, {v: k.split(":", 1)[1] for k, v in _mapping.items()}
```

这段代码里的 `_mapping` 和 `_counter` 放在模块作用域只是为了看得清楚。生产环境里必须按用户会话隔离，存进 Redis，TTL 与会话生命周期一致，并且 **绝不能写进应用日志**。最后这条是假名化最常见的失败方式：某位工程师在一个看着无害的地方加了 `logger.info(f"session_state={session}")`，映射就进了日志聚合系统，于是你的调试日志里躺着可还原的 PII。映射是整个系统里最敏感的产物。请按这个级别对待它。

一个更隐蔽的坑：`<PERSON_1>` 这类占位符有时会让模型犯迷糊，它会拒绝“给占位符发邮件”。解决办法是把带类型的标签换成看着像真人的合成姓名（Faker 库），反向映射只保留在内部：对模型来说 `Alice Smith` 变成了 `Jordan Park`，对用户来说 `Jordan Park` 还是 `Alice Smith`。Presidio 在 `docs/samples/python/pseudonymization/` 的示例演示了这个模式。

**带 ZDR 的透传** 把原始 PII 发给服务商，靠的是服务商合同上的零数据保留（zero data retention，ZDR）。它不是脱敏的替代品。服务商在传输过程中照样看得到数据，滥用监控系统可能标记并保留它，你自己的日志也可能仍然记下它。ZDR 应当作为脱敏或假名化之上的一层，而不是替代。

![一张三栏横向示意图，同一个用户输入分别流过三种架构：带 ZDR 的透传把原始文本直接送给模型；脱敏在调用前剥掉 PII 片段且没有回程路径；假名化在调用前把 PII 换成 token，并在返回时通过会话存储换回原文，其中会话存储以珊瑚色突出显示，标注为最敏感的组件](../../../ai-eng/part-9-security-and-governance/images/04-three-architectures.png)
*同一份输入，三种架构。假名化里的会话存储是系统里最敏感的一块，比模型 API 本身还敏感。*

## 服务商的保留策略是按功能划分的，不是组织级

你在销售材料里看到的那个醒目的保留期限数字，很少覆盖你应用实际用到的所有 API 功能。下面是三家主要服务商当前的真实规则，全部标注“截至 2026 年 6 月”，因为政策变动很频繁。

**OpenAI API。** 标准 API：输入和输出最多保留 30 天用于滥用监控，之后删除，法律要求保留的除外。零数据保留（ZDR）：只在符合条件的端点上提供，需要通过销售申请，用途还得合格。欧洲数据驻留（data residency）项目类型（2025 年 2 月上线，到 2025 年 11 月扩展到覆盖欧盟、英国、美国、加拿大、日本、韩国、新加坡、澳大利亚、印度和阿联酋）把区域内处理与 ZDR 打包，面向符合条件的 API 客户。默认不用 API 数据训练模型。[^3][^4]

**Anthropic Messages API。** ZDR 在 Messages API 和 Token Counting API 上是*默认*行为。响应返回后，提示词和输出不做静态保留。只有 KV cache 的表示会留在内存里，活到缓存 TTL 结束。[^5] 但是：Batch API 有 29 天保留期，不符合 ZDR 资格。Files API 保留到你显式删除。Code Execution 容器最多保留 30 天。Claude Managed Agents 会持续保存会话记录，直到被删除。MCP connector 同样不符合 ZDR 资格。如果你的架构用到其中任何一项，就不能只靠 ZDR。

第二个容易让团队翻车的例外：Claude Fable 5 和 Claude Mythos 5 属于“Covered Models”，强制 30 天保留。它们不支持 ZDR。已经签了 ZDR 安排的组织，必须把某个工作区切到 30 天保留，才能用这两个模型。[^5] 另外，无论哪种方案，一旦 Anthropic 因违反使用政策而标记了内容，即使开了 ZDR 也可能保留最多 2 年。

Anthropic 的数据驻留分成两个独立开关。`inference_geo` 参数（按请求设置，取值 `"us"` 或 `"global"`）控制请求在哪里处理。工作区地理（创建时设定，目前只提供 `"us"`）控制静态存储的位置。截至 2026 年 6 月，仅限美国推理按标准价格的 1.1 倍计费，且只有 Claude Opus 4.6、Sonnet 4.6 及之后的版本支持该参数。更老的模型会返回 400。如果你的错误处理在遇到 400 时会剥掉未知参数再重试，那你已经悄悄把 PHI 路由到了全球基础设施上。改为在工作区层面设置 `allowed_inference_geos: ["us"]` 策略，让请求直接报错失败。[^6]

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    inference_geo="us",
    messages=[{"role": "user", "content": "Summarize this document."}],
)

assert response.usage.inference_geo == "us"
```

**Azure OpenAI / Foundry。** 三类部署：Global（任意 Azure 区域）、DataZone（仅美国或欧盟数据区）、Standard/Regional（指定单一区域）。截至 2026 年 5 月，DataZone EU SKU 覆盖法国、德国、意大利、荷兰、挪威、波兰、西班牙、瑞典和瑞士。[^7] 部署类型在创建资源时选定，之后改不了。还要注意：即便是 DataZone，默认也会把一些支撑服务（遥测、日志、审核）路由到区域之外。要完全隔离，需要额外的 Azure 资源配置，并对 Microsoft 的 EU Data Boundary 承诺单独做一次评估。

一条好用的判断规则。如果你需要服务商侧接近零保留、而且只想调一个开关，Anthropic Messages API 配 ZDR 是最简单的合同故事，因为那就是默认。OpenAI 要创建欧洲项目，或者通过销售谈 ZDR。Azure 要在最开始就选对部署 SKU。

## 合同链条有四层深

按 GDPR 第 4(7) 与 4(8) 条，用户所在公司是控制者（controller），你的应用是处理者（processor，或共同控制者），模型 API 服务商是次级处理者（subprocessor），而 API 服务商的云厂商或审核外包商是次次级处理者（sub-subprocessor）。欧洲数据保护委员会（EDPB）的 Opinion 22/2024（2024 年 10 月）是当前权威指引。对工程师最关键的结论是：控制者不必系统性地逐份索取次级处理合同，但义务没有沿链条传递下去时，责任在控制者身上。[^8]

实际需要核验三件事。

第一，你和模型 API 服务商之间有 DPA。OpenAI 的 DPA 在你接受 Commercial Terms 时并入；Anthropic 带 SCC 的 DPA 是其 Commercial Terms of Service 的一部分。Azure 走 Microsoft 的 Product Terms。[^4][^9] 这一项就算过。但如果你是经过第三方 SaaS 访问 Claude（比如一个集成了 Claude 的 CRM），管你的就不是 Anthropic 的 DPA，而是那个第三方平台的条款。你需要和真正签约的那一方有 DPA。

第二，次级处理者清单是最新的、可接受的。OpenAI 2025 年 4 月的清单列了 Cloudflare（CDN）、Microsoft（云，多区域）、Snowflake（数据仓库，美国）、TaskUs（内容审核，菲律宾）、Intercom（客户支持，美国）、Salesforce（多区域）、Accenture（加拿大、菲律宾）、Confluent（美国）、Cinder Technologies（美国）、WorkOS（美国）、Okta/Auth0（美国）。[^10] 星号脚注很重要：其中大多数在 ZDR 生效时并不适用。订阅变更通知列表。第 28(2) 条要求次级处理获得一般授权或特别授权，服务商默认采用一般授权模式，变更通过邮件通知你。

第三，跨境传输有合法性基础。SCC（Standard Contractual Clauses，标准合同条款）是 EEA 向第三国传输的默认机制。OpenAI 在关联公司之间使用 SCC（OpenAI Ireland 是 EEA 的签约主体）。Anthropic 的 DPA 涵盖 SCC。[^9][^10] 多数工程师不读 SCC。你也不必读。但你得确认它们存在。

## ZDR 解决不了 CORS，以及其它实操中的意外

开了 ZDR 之后，有几件运维层面的事会让团队吃苦头。

ZDR 之下不支持 Anthropic 的 CORS。[^5] 你不能从浏览器端的 JavaScript 直接调 API。必需的架构是：一个服务端代理，代表前端去调 Claude。这多了一跳，也多了一个要维护的后端服务，但同时也让你的 API key 不出现在浏览器代码里，这本身是另一项安全收益。

LiteLLM 把 Presidio 集成做成了前置请求钩子：代理拦截每一次出站 LLM 调用，把提示词送给 Presidio sidecar，匿名化其中的片段，再转发脱敏后的提示词。微软为这个模式提供的参考部署在 `microsoft.github.io/presidio/samples/docker/litellm/`。[^1] 同样的模式也能挂到响应侧，处理那些把检索内容里的 PII 又吐回来的输出。

对 HIPAA，Anthropic 和 OpenAI 都提供覆盖 Messages API 的 BAA（Anthropic 还覆盖 Token Counting）。Anthropic 明确把 Code Execution、Files API、Claude Managed Agents 和 Claude Code 排除在 BAA 范围之外。[^5] 即使 BAA 签了，也不要让 PHI 走任何不受支持的功能；BAA 根本不覆盖那条数据路径。

## 最容易做错的几个决定

一份短清单，按在复盘里出现的频率排序。

- **映射进了日志。** 假名化保护的是模型 API，不保护你自己的日志聚合系统。审计一下日志代码里有没有叫 `mapping`、`entity_map` 或 `session_state` 的字段。
- **以为 ZDR 是组织级的。** 把你应用发出的每一个 API 调用，对照服务商的功能资格表过一遍。Files API、Batch API 和 Code Execution 都会打破这个假设。
- **次级处理者清单只在签约时看过一次。** 订阅变更通知，按期复查。新的次级处理者会出现，区域会变，而 GDPR 的责任是你的。
- **推理地理静默地失败开放（fail open）。** 老模型拒绝这个参数；粗糙的重试逻辑把它剥掉；PHI 就这么走上了全球基础设施。改成用工作区策略直接把请求拒掉。
- **检测模式漏掉真实世界的格式。** 拿你自己的真实数据测 Presidio。写成 `078051120`（不带分隔符）的美国 SSN 会漏掉默认模式。扩展识别器，或者加一条 deny-list 关键词，在上下文里给它提分。

下一章 [合规与审计](../../../ai-eng/part-9-security-and-governance/05-compliance-and-audit.md) 讲审计员到底想在你的日志里看到什么，以及这会怎样影响你现在正在配置的保留策略。

## 参考文献

[^1]: Microsoft, "Presidio: Data Protection and De-identification SDK," accessed June 2026, https://microsoft.github.io/presidio/

[^2]: Microsoft, "PII entities supported by Presidio," accessed June 2026, https://microsoft.github.io/presidio/supported_entities/

[^3]: OpenAI, "Enterprise privacy at OpenAI," updated 8 January 2026, https://openai.com/enterprise-privacy/

[^4]: OpenAI, "Introducing data residency in Europe," 5 February 2025 (with updates through January 2026), https://openai.com/blog/introducing-data-residency-in-europe

[^5]: Anthropic, "API and data retention," accessed June 2026, https://docs.anthropic.com/en/docs/build-with-claude/zero-data-retention

[^6]: Anthropic, "Data residency," accessed June 2026, https://docs.anthropic.com/en/docs/build-with-claude/data-residency

[^7]: Microsoft, "Deployment types for Microsoft Foundry Models," updated 27 February 2026, https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/deployment-types

[^8]: European Data Protection Board, "EDPB adopts Opinion on processors," 9 October 2024, https://www.edpb.europa.eu/news/news/2024/edpb-adopts-opinion-processors-guidelines-legitimate-interest-statement-draft_nb

[^9]: Anthropic, "How do I view and sign your Data Processing Addendum (DPA)?", 16 March 2026, https://support.anthropic.com/en/articles/7996862-how-do-i-view-and-sign-your-data-processing-addendum-dpa

[^10]: OpenAI, "OpenAI Sub-processor List," effective 30 April 2025, https://openai.com/policies/sub-processor-list-april-2025-update/
