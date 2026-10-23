---
title: "合规与审计"
description: "面对 SOC 2 和欧盟 AI 法案，工程师真正要建的东西：结构化审计留痕、防篡改可验证的保留机制，以及监管方会去找的模型文档。"
slug: 05-compliance-and-audit
part: 9
chapter: 5
difficulty: advanced
prerequisites:
  - 04-pii-and-privacy
  - part-8-evaluation-and-observability/06-tracing
tags: [llm, ai-safety]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# 合规与审计

2026 年，一家金融科技团队为 KYC 上线了一个文档分类 agent，随后收到 FCA 审查员的一封邮件。“把你们 3 月 14 日标记的那本护照的完整审计留痕（audit trail）调出来。”他们的日志里只有最终那段 JSON 响应，没有别的。为了还原这个答案，他们翻了四天的 git blame 和 CI 产物。审查员给了两周：要么做出真正的审计系统，要么暂停这条流水线。[^1]

一段话就能说完 AI 功能的合规与审计。法规（欧盟 AI 法案、SOC 2 信任服务准则）规定的是哪些事必须可被证明：决策可追溯、系统动作有日志、模型行为有文档。工程要做的是让这些证明成为可能的数据结构、保留策略（retention policy）和埋点（instrumentation）。AI 系统让这件事比传统软件更难，因为同一份输入，经过略微不同的模型版本、提示词修订或检索结果，就可能产生产出不同的输出。如果你还原不出产生某个决策的那套配置，你也就解释不了那个决策。

## 欧盟 AI 法案到底要求工程师做什么

该法案（Regulation EU 2024/1689）于 2024 年 8 月 1 日生效，义务按六个日期分批落地。[^2] 2026 年对应用工程师真正重要的日期是：

- **2025 年 2 月 2 日。** 禁止性做法被禁（第 5 条）。AI 素养义务（第 4 条）。
- **2025 年 8 月 2 日。** 针对基础模型提供者的通用 AI（GPAI）义务（第 53-55 条）。AI Office 开始运作。
- **2026 年 8 月 2 日。** 高风险 AI 系统（附件 III）：记录保存（第 12 条）、自动生成的日志（第 19 条）、技术文档（第 11 条）、人工监督（第 14 条）。
- **2027 年 8 月 2 日。** 2025 年 8 月之前投放市场的存量 GPAI 模型。
- **2027 年 12 月 2 日。** 关键基础设施里的高风险 AI（生物识别、边境管控）。
- **2028 年 8 月 2 日。** 受监管产品里的高风险 AI（医疗器械、机器人、机械）。

非欧盟市场的工程师会犯两个错误。第一：法案适用于在欧盟*使用*的系统，不管提供者或部署者注册在哪里。第二：法案区分*提供者*（provider，构建系统并把它投放市场）与*部署者*（deployer，在自己的业务里使用它）。做 AI 简历筛选工具的创业公司是提供者，接入这个工具的银行是部署者。提供者的义务更重，但两者都有义务。

如果你通过 API 接入 GPT-4o、Claude 或 Gemini，你就是某个 AI 系统的下游提供者，而 GPAI 提供者必须按附件 XII 规定的规格把技术文档交给你，你才能履行自己的义务。截至 2025 年 8 月，这项义务已经生效。[^3]

大多数 LLM 聊天机器人不属于高风险。通用客服机器人几乎肯定不是。而自动批准或拒绝信贷、给简历打分、给学生评分、做生物特征比对的系统，就是高风险。附件 III 列举了这些类别，第 6 条管分类逻辑，包括在什么条件下某个高风险功能会因为最终由人拍板而掉出适用范围。陷阱在于：分类是你的责任。如果你上线了某个高风险的东西，却没写下“为什么我判定它不是高风险”，监管方就可以自己下判断，而你毫无辩护余地。在功能上线时写好那份两段式的分类备忘，点名你考虑过的附件 III 类别，解释你的功能为什么落在或不落在其中；功能范围一变，就回头重看一遍。

罚款是真的。GPAI 违规：最高 1500 万欧元或全球营业额的 3%（第 101 条）。高风险系统要求：最高 1500 万欧元或 3%（第 99(3) 条）。禁止性做法：最高 3500 万欧元或 7%（第 99(2) 条）。[^4]

## 一条审计记录里到底有什么

AI 审计日志与普通应用日志有三点不同。它记录决策那一刻的配置状态（模型版本、提示词哈希、temperature），而不只是输入和输出。它是只追加（append-only）、防篡改可验证的（tamper-evident）。而且非工程人员也能按单条决策去查询它。

一条高风险决策的最小 schema（来自那些已经跟监管打过一轮交道的生产部署）：[^1][^5]

```python
import hashlib, json, time
from dataclasses import dataclass, asdict

@dataclass
class AuditRecord:
    run_id: str
    timestamp_start: float
    timestamp_end: float
    model: str               # 精确锁定的版本
    prompt_rev: str          # 提示词文件的 git commit
    input_hash: str          # 规范化 JSON 的 SHA-256
    tool_calls: list         # 有序账本
    retrieved_chunk_ids: list
    decision: str
    rationale: str
    user_id_hash: str        # 不是原始 PII

def make_input_hash(payload: dict) -> str:
    canonical = json.dumps(payload, sort_keys=True, ensure_ascii=True)
    return "sha256:" + hashlib.sha256(canonical.encode()).hexdigest()

def make_user_hash(user_id: str, salt: str) -> str:
    return "sha256:" + hashlib.sha256(f"{user_id}:{salt}".encode()).hexdigest()
```

每个字段都是为了回答监管方或审计员一定会问的某个具体问题而存在的。`model` 和 `prompt_rev` 合起来，让你能还原产生这个决策的那套配置。`input_hash` 证明输入从进来到决策之间没被改过，同时不必存原始 PII（那会跟 GDPR 的数据最小化原则冲突）。`tool_calls` 是执行图：每一次工具调用、它的参数、结果的 SHA-256、延迟和重试次数，按顺序排好。`retrieved_chunk_ids` 让你能指到模型当时用的那版策略或文档。`user_id_hash` 让动作可归因，又不把用户身份泄进日志。

团队犯的最大错误是只记最终响应。当调查员问“模型为什么产出这个输出”，团队答不上来，因为中间状态从来没被持久化。如果仅凭那一条日志记录重建不出决策过程，这个缺口就在你身上。在任何受监管场景上线之前，先把审计记录做进去。事后补装要难得多：提示词版本会丢，模型历史会变得说不清，早期的决策直接不可恢复。

第二大错误是记原始 PII。工程师默认什么都记，好方便调试；隐私分析要么往后拖，要么永远不做。结果就是一个存满原始 PII 的库，它本身就违反 GDPR 第 5(1)(e) 条（存储限制）和第 25 条（默认数据保护）。AI 法案第 19 条明确交叉引用了欧盟个人数据法。含 PII 的字段记输入的*哈希*，用户记成假名哈希；敏感的输出文本单独放一层，用更短、目的受限的保留策略，跟那个 6 个月的合规层分开。

## 防篡改存储

第 19 条要求提供者把第 12 条日志至少保留 6 个月，其他欧盟法律要求更久的按更久的来（比如金融服务）。只追加、防篡改可验证的存储，才让你能对审计员说：他们现在读到的记录，就是你当初写下的记录。

标准做法分两层。热存储放在可查询的数据库里（Postgres 加行级安全策略，禁止对审计表 UPDATE 和 DELETE；分析型负载用 ClickHouse），保留最近 30 到 90 天。冷存储放 S3，开 Object Lock 的 COMPLIANCE 模式，保留 183 天以上。COMPLIANCE 模式（不是 GOVERNANCE）在保留期到期前谁也删不掉，包括 root 账号——正是这个性质让你能面不改色地说“这条记录是防篡改可验证的”。

```python
import boto3

def create_tamper_evident_bucket(bucket_name: str, region: str = "us-east-1"):
    s3 = boto3.client("s3", region_name=region)
    s3.create_bucket(
        Bucket=bucket_name,
        ObjectLockEnabledForBucket=True,
    )
    s3.put_object_lock_configuration(
        Bucket=bucket_name,
        ObjectLockConfiguration={
            "ObjectLockEnabled": "Enabled",
            "Rule": {
                "DefaultRetention": {
                    "Mode": "COMPLIANCE",
                    "Days": 183,
                }
            },
        },
    )
    return bucket_name
```

成本上，一条带六次工具调用的典型 agent 决策 trace 大约 12 KB。一百万条决策放冷存储大约 12 GB。[^1] 这个量级下，保留成本跟推理成本比就是预算里的零头。不要在这一点上做优化。

![一张两层存储的示意图：左侧是热层，标注为 Postgres 只追加，带一个 30 到 90 天保留期的标签；一条粗箭头标注 nightly export 连到右侧的冷层，标注为 S3 Object Lock COMPLIANCE，带一个用珊瑚色高亮的 183 天保留期标签，作为防篡改层；两层都汇入顶部标注为 compliance reviewer 的查询 API](../../../ai-eng/part-9-security-and-governance/images/05-retention-tiers.png)
*两层，两个保留窗口。S3 那把锁才是让你能面不改色说“防篡改可验证”的那一层。*

## AI 功能怎么过 SOC 2

截至 2026 年 6 月，还没有专门针对 AI 的 SOC 2 扩展。AI 功能跟你其余软件用同一套通用准则（Common Criteria）来评。受影响最大的五条：[^6]

- **CC6.1（逻辑访问）。** AI 系统常有非人身份：带 API key 的 LLM agent、服务账号。审计员查的是这些身份是否跟人类用户受同一套治理：最小权限、密钥轮换、吊销流程。把每个 LLM agent 身份都当成特权服务账号对待。90 天轮换、权限范围收窄、agent 下线时自动吊销。多数团队漏掉这条。
- **CC6.3（离职时的权限回收）。** 会累积权限的 AI agent，尤其是用户授予的 OAuth token，需要明确的吊销流程。用户一走，下游 agent 的权限也跟着收。
- **CC7.1（系统运维与异常检测）。** 对 AI 输出做持续监控。单次调用延迟、错误率、输出分布漂移、token 用量尖峰。审计员看的是告警阈值、runbook、成文的应急响应。
- **CC7.2（组件异常监控）。** 模型输出质量下降也算异常。生产评估流水线（见[在线评估与 A/B 测试](../part-8-evaluation-and-observability/05-online-evaluation-and-ab-testing.md)）只要能产出带评分的记录，就能同时当 CC7.2 的证据用。
- **CC9.2（供应商风险）。** 每一个第三方模型 API 都是一段供应商关系。你需要 DPA、供应商挂掉时的成文兜底方案，以及审阅供应商自己的 SOC 2 Type II 报告。截至 2026 年，OpenAI、Anthropic 和 Google Cloud 都持有 SOC 2 Type II，并通过各自的信任门户向客户提供报告。[^7]

能把一切聚焦起来的那个审计提问：“给我看你们的 AI 在 3 月 14 日对那个用户说了什么。”合格的回答只有两种。一种是“记录在这儿”。另一种是“我们在系统设计上就让这个问题问不出来，这里是当时的设计决策文档”。介于两者之间的任何回答，都是一条审计发现（finding）。[^8]

## 模型卡与系统卡

模型卡（model card，Mitchell 等，FAT* 2019）是随训练好的机器学习模型一起发布的短文档：预期用途、评估方法、在不同人群上的表现、局限、超出范围的使用。[^9] 系统卡（system card）把它扩到整个已部署系统，包括系统提示词、护栏、检索配置和工具访问权限。OpenAI 的 GPT-4o 和 o3 系统卡是标准范例；Anthropic 和 Google 每次发布模型都出模型卡。[^10]

从 2025 年 8 月起，欧盟 AI 法案把原本自愿的事对 GPAI 提供者变成了强制。附件 XI 第 1 节要求每个 GPAI 提供者记录：预期任务、可接受使用政策、发布日与分发方式、架构、参数规模、输入输出模态、许可证、训练过程的设计选择与方法、数据来源与清洗、训练时长、算力（FLOPs）、能耗。第 2 节（针对有系统性风险、用超过 10^25 FLOP 训练的 GPAI）追加对抗测试结果和完整系统架构。[^3]

附件 XII 规定 GPAI 提供者必须向下游集成方提供什么：模型能力与局限、上下文长度、输出格式、已知风险与安全措施、安全接入说明。作为接入 GPT-4o 或 Claude 的应用工程师，这就是你上线时应该能指出来的那份文档。截至 2026 年 6 月，主流提供者大多已公开发布详细的模型卡，但严格按附件 XII 结构来的还不太一致。要盯住哪份文档对应哪项义务。

一个不容易察觉的坑：在部署清单、审计日志和系统卡里都存精确的模型版本串（`gpt-4o-2024-11-20`，而不是“GPT-4o”）。加一条 CI 检查，部署版本跟文档版本不一致就失败。法案要求文档保持最新；一张跟已部署系统对不上的过期卡片，本身就是不合规。

2023 年一项对 32,000 张 Hugging Face 模型卡的系统分析，在“哪些部分会被跳过”上给了个很有启发的结论：环境影响、局限、评估这三节的填写率最低，训练那一节完成得最齐。[^11] 监管方最在意的字段，恰好是工程师最常留空的字段。

## 真正现成的工具

这些你都能用 Postgres + S3 自己搭。多数团队第一版就是这么做的。当你需要一层托管的可观测层时，开源的默认选择是 Langfuse：trace 存在 ClickHouse，trace–observation–score 的层级数据模型对齐 OpenTelemetry 的 GenAI span，企业版还有针对管理动作的审计日志（谁改了哪个提示词、谁删了哪条 trace，带完整的前后状态）。[^12][^13] 他们公开的架构（2024 年 12 月）解决的是生产里的常见难题：agent 工作流里一条 trace 带几千个嵌套 observation。

供数给这一切的埋点层在[链路追踪](../part-8-evaluation-and-observability/06-tracing.md)里讲。你发出去的是 OpenTelemetry 的 GenAI span 模型；这一章讲的是包在它外面的保留、分类和文档。

## 本周做什么

如果你面向企业或受监管行业交付，下一次合规评审之前，有四件事没有商量余地：

- 每条决策一条审计记录，含上面那九个字段，并且在响应返回给用户之前就写好。
- 两层保留方案：热的放在业务数据库里 30 到 90 天，冷的放 S3 Object Lock COMPLIANCE 至少 183 天。
- 跟每个模型 API 提供者签好 DPA，并存一份他们最新的 SOC 2 Type II 报告。
- 每个 AI 功能一份两段式的欧盟 AI 法案分类备忘，点名考虑过的附件 III 类别，以及该功能算或不算高风险的理由。

红队（red-teaming）这门功夫产出的正是“护栏真的管用、审计日志真的抓到了它声称抓到的东西”这类证据，下一章 [红队](./06-red-teaming.md) 接着讲。

## 参考文献

[^1]: Isaac Sinclair, "The Five Fields Your Agent Audit Log Is Missing," Medium, May 2026, https://medium.com/@isaacsinclair90/the-five-fields-your-agent-audit-log-is-missing-99a48a98024a

[^2]: Regulation (EU) 2024/1689 of the European Parliament and of the Council (EU AI Act), Official Journal, 13 June 2024, https://artificialintelligenceact.eu/the-act/

[^3]: European Commission, "General-purpose AI obligations under the AI Act," last updated 1 August 2025, https://digital-strategy.ec.europa.eu/en/factpages/general-purpose-ai-obligations-under-ai-act; "Annex XI," https://artificialintelligenceact.eu/annex/11/

[^4]: ai-act-law.eu, "Article 101: Fines for providers of general-purpose AI models," https://ai-act-law.eu/article/101/

[^5]: artificialintelligenceact.eu, "Article 12: Record-Keeping" and "Article 19: Automatically Generated Logs," EU AI Act, https://artificialintelligenceact.eu/article/12/, https://artificialintelligenceact.eu/article/19/

[^6]: AICPA, "SOC 2 Trust Services Criteria," 2017 (with 2022 updates), https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2/

[^7]: OpenAI, "Security and privacy" and Trust Portal, https://openai.com/security-and-privacy/ and https://trust.openai.com/; Anthropic, "What Certifications has Anthropic obtained?", March 2026, https://support.claude.com/en/articles/10015870-what-certifications-has-anthropic-obtained; Google Cloud, "SOC 2", https://cloud.google.com/security/compliance/soc-2

[^8]: localaimaster.com, "Local AI Audit Trail: Log Every Prompt & Response (2026)," https://localaimaster.com/blog/local-ai-audit-trail

[^9]: M. Mitchell et al., "Model Cards for Model Reporting," FAT* 2019, arXiv:1810.03993, https://arxiv.org/abs/1810.03993

[^10]: OpenAI, "GPT-4o System Card," 2024, https://openai.com/index/gpt-4o-system-card/

[^11]: "What's documented in AI? Systematic Analysis of 32K AI Model Cards," arXiv:2402.05160, 2023, https://arxiv.org/html/2402.05160v1

[^12]: Langfuse, "Audit Logs," documentation, https://langfuse.com/docs/administration/audit-logs

[^13]: Langfuse, "From Zero to Scale: Langfuse's Infrastructure Evolution," December 2024, https://langfuse.com/blog/2024-12-langfuse-v3-infrastructure-evolution
