---
title: "红队测试"
description: "自动化探测套件、人工测试流程，以及 CI 中的持续红队测试：在被别人攻破之前，先自己攻一遍。"
slug: 06-red-teaming
part: 9
chapter: 6
difficulty: advanced
prerequisites:
  - 00-prompt-injection
  - 02-guardrails
  - part-8-evaluation-and-observability/04-regression-suites-and-ci-gates
tags: [llm, ai-safety]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 红队测试

2024 年 4 月到 5 月，微软研究团队用一种叫 **Skeleton Key** 的多步越狱（jailbreak）手法，测了七个前沿模型：Llama 3 70B Instruct、Gemini Pro、GPT-3.5、GPT-4o、Mistral Large、Claude 3 Opus 和 Cohere Commander R Plus。这套攻击的做法是请模型*增强*而不是改变它的行为准则，把配合说成科学研究，只要求在有害输出前面加一句警告。七个模型全部配合。只有 GPT-4 表现出部分抵抗，而且仅在系统消息与用户输入被严格隔离时才成立。[^1]

红队测试（red-teaming）存在的理由就在这里。投入了大量安全资源的模型，照样会栽在标准安全基准里根本没出现的多轮攻击上。你的任务是在自己的部署里、用自己的系统提示词和工具，抢在攻击者之前把这些失败找出来。微软从 2024 年 10 月前对 100 多个生成式 AI 产品做红队测试得出的那条结论，值得当作基本框架：“generative AI amplifies existing security risks and introduces new ones”——生成式 AI 会放大已有的安全风险，还会引入新的风险，包括那些与 AI 层无关、独立存在的经典 SSRF 和注入类漏洞。[^2]

## 自动化探测套件能给你什么

有三款像样的工具覆盖自动化这条“广度”路线。它们互为补充而非替代品，各自对应不同的活儿。

**garak** 是模型层面的广度扫描器，最初出自 NVIDIA，现在由 github.com/NVIDIA/garak 维护，Apache 2.0 许可（截至 2025 年 3 月，PyPI 上是 0.10.3 版）。[^3] 一套 harness 里有四个部件：Generators（任意 LLM 目标）、Probes（按漏洞类别组织的对抗提示词集合）、Detectors（关键词与 ML 分类器两种输出评估器）、Buffs（发送前施加的模糊变换）。探测库覆盖提示词注入（PromptInject、含 666 条社区越狱样本的 DAN 变体、AutoDAN、GCG）、训练数据回放、基于编码的绕过（14 种编码：base64、ROT13、UU、盲文、摩尔斯、Zalgo、北约音标）、包名幻觉、通过 Markdown 图片标签触发的 XSS，以及面向多模态模型的视觉越狱。[^4]

atkgen 模块会在那些吐出过有害内容的对话轮次上微调一个小模型，再拿这个攻击者模型去打目标模型。作为基线：面对 GPT-3.5，atkgen 能在 1.0% 的轮次里诱出有害内容；面对 OPT 6.7B 是 26.7%；面对 GPT-2 是 17.0%。[^4] 经过安全微调的前沿模型对通用攻击的抵御水平在低个位数的攻击成功率（attack success rate，ASR），而微调较少的模型失败得频繁得多。

**PyRIT**（微软，2024 年 2 月发布，截至 2026 年 6 月为 0.14.0 版）是多轮自适应攻击框架，“battle-tested on 100+ products including Copilot.”[^5][^6] 它最强的地方是内置攻击库：Crescendo（多轮升级，直到最后一轮都停留在良性表述空间里[^7]）、Skeleton Key（那项七模型研究里的“增强而非改变”套路）、TAP（Tree of Attacks with Pruning，用一个攻击者 LLM 在树结构里生成并剪枝攻击变体）。PyRIT 配有 SQLite 或 Azure SQL 存储，能跨多次运行保留对话记录和评分。CoPyRIT 图形界面让不写代码的红队人员也能跑标准化场景。

**promptfoo**（2025 年被 OpenAI 收购）是 YAML 优先、映射到合规框架的那个选项。[^8] 它的 red-team 模块按六大类提供 157 个插件，并对 `owasp:llm:01` 到 `owasp:llm:10`、`mitre:atlas:reconnaissance`、`nist:ai:measure:1.1`、`iso:42001`、`eu:ai-act`、`gdpr` 这些标识符做了一等公民级的映射。发现项自带监管分类标签，省掉了合规团队后面的一大摊活。对多数生产部署来说，真正相关的是面向 agent 的那组插件：`prompt-injection`、`indirect-prompt-injection`、`pii:direct`、`rbac`、`bola`、`bfla`、`ssrf`、`agentic:memory-poisoning`、`data-exfil`、`rag-poisoning`。策略（payload 怎么送进去）是另一码事：`jailbreak`、`crescendo`、`multi-turn`、`base64`、`rot13`、`leetspeak`。

一个实用的分层：上新模型版本之前，用 garak 做基线模型层扫描；用 promptfoo 跑映射到 OWASP 的 CI 套件；用 PyRIT 跑每晚执行的自适应多轮攻击。这三者单独都覆盖不了产品层面的攻击面（代码执行、访问控制、Web 应用漏洞）。微软的发现里就包括一个视频处理类 AI 因依赖过期的 FFmpeg 而中招的经典 SSRF。[^2] AI 专属的探测抓不到这种。常规的 Web 安全扫描要和 AI 扫描一起跑。

## 用 OWASP 和 ATLAS 定义范围词汇

两套分类体系撑起红队测试的范围。它们互补，不重复。

**OWASP LLM Top 10（2025）** 用来给风险排序、定优先级。2025 版（2024 年 11 月发布）重编了几个条目的编号，并新增两类：系统提示词泄露（LLM07）和向量与嵌入弱点（LLM08）。LLM10 从 “Model Denial of Service” 改名为 “Unbounded Consumption”，以覆盖“掏空钱包”式攻击，而不只是可用性被拖垮。[^9] 完整清单：提示词注入、敏感信息泄露、供应链、数据与模型投毒、输出处理不当、过度代理权限（Excessive Agency）、系统提示词泄露、向量与嵌入弱点、虚假信息、无界消耗。[^10]

**MITRE ATLAS** 描述攻击者实际会做什么（技术词汇表）。14 个战术类别、80 多项技术，沿用与 ATT&CK 相同的杀伤链结构。[^11] LLM 红队测试相关的标识符有：AML.T0051（LLM Prompt Injection，直接注入）、AML.T0054（Indirect Prompt Injection，间接注入）、AML.T0020（Poison Training Data）、AML.T0019（Publish Poisoned Datasets）、AML.T0048（Compromise ML Software Dependencies）、AML.T0024（Exfiltration via ML Inference API）、AML.T0034（Cost Harvesting / 掏空钱包）、AML.T0043（Craft Adversarial Data）。

OWASP 给你优先级，ATLAS 给你贴在每条发现上的技术 ID。提示词注入（LLM01）横跨两个战术阶段，对应三条不同的 ATLAS 技术。两个都用。promptfoo 把两者都接进了插件标识符，garak 的 HTML 报告按 OWASP 类别分组。写发现报告时，两个编号都引。

## CI 里的三个层级

持续红队测试意味着每次代码变更都跑探测，一旦出现新漏洞就让流水线失败或告警。能落地的架构长这样：

- **第 1 层（每个 PR）。** 20 到 50 条快速、确定性的探测，覆盖提示词抽取、系统提示词覆盖、RBAC、跨会话泄露。墙钟时间控制在 2 分钟以内。工具：promptfoo 配一组固定的小插件集，不做远程推理。目标：抓回归。门禁条件：上次构建能过、这次失败的探测。
- **第 2 层（合入 main 时）。** 全量插件套件，范围对准你应用对应的 OWASP 与 ATLAS 攻击面。视供应商限流情况，5 到 30 分钟。工具：garak 或 promptfoo 配远程推理。目标：对已合入代码做全覆盖。门禁条件：第 1 层那条回归规则，外加任何首跑就失败的新探测类别。
- **第 3 层（每晚或发布前）。** 自适应攻击生成、多轮 Crescendo 与 TAP、领域专家评审。工具：PyRIT 加人工评审。目标：发现新攻击。不设硬门禁，结果进待办池。

```python
import subprocess

def run_garak_scan(model_type: str, model_name: str, probes: str = "all"):
    cmd = [
        "python", "-m", "garak",
        "--model_type", model_type,
        "--model_name", model_name,
        "--probes", probes,
        "--report_prefix", f"scan_{model_name.replace('/', '_')}",
    ]
    return subprocess.run(cmd, capture_output=True, text=True)
```

真正要紧的 CI 失败规则是：PR 之前就已经在失败的探测，*不算*阻塞 PR 的失败。那是单独跟踪的历史欠账。按绝对通过率卡门禁，等于让所有人替历史债务受罚；按回归卡门禁，流水线才诚实。promptfoo 原生支持 GitHub Actions，`npx promptfoo redteam run`，阈值可配，超阈值时非零退出。garak SDK（截至 2026 年 5 月，PyPI 上的 `garak-sdk`）提供编程接口，可以不用 CLI 就把扫描嵌进 CI。[^12]

GoDaddy 的 Nemesis 系统（工程博客，2026 年 5 月）描述的正是这个模式：每晚自动跑红队，产出每日记分卡，其中有一个设计决定值得照抄——提示词漂移检测。[^13] 探测用例的生成往往以系统提示词为条件。系统提示词一改而探测不重新生成，套件照样全绿，因为它测的是上一版提示词。Nemesis 对系统提示词做哈希，哈希一变就自动触发对抗测试用例的重生成。正如 GoDaddy 所说：一套在系统提示词更新一次之后就过期的测试套件，只是一种虚假的安全感。

![一张纵向泳道图，三条横带自上而下堆叠，分别代表第 1 层“每个 PR”，左侧标出 2 分钟时间预算；第 2 层“合入 main”，预算 5 到 30 分钟；第 3 层“每晚”，不限时。每条带标注所用工具，带与带之间由一个三角形的通过—失败门禁符号分隔，其中第 1 层的门禁用珊瑚色高亮，因为它是触发最频繁的回归阻断点](../../../ai-eng/part-9-security-and-governance/images/06-three-tier-ci.png)
*三个层级，三份时间预算，三种职责。最要紧的那道门禁，反而是最小的那道。*

## 自动化止步、人接手的地方

自动化管广度，人管深度。微软从 100 多个产品里得出的结论很直白：红队测试无法完全自动化。[^2] 最有力的证据相当朴素：判断输出里有没有仇恨言论或露骨色情内容，LLM 已经足够可靠；判断医药、网络安全、CBRN 或文化特定伤害，它就不可靠。领域专家能抓到自动化工具系统性漏掉的失败。文化理解力能抓到那些用英文语料训练出来的分类器一路放行的多语言探测。

微软的人工流程分五个阶段。界定并分析系统（它做什么、部署在哪、谁在用、适用哪些 OWASP 与 ATLAS 类别）。建威胁模型（对抗角色、TTP、系统弱点、下游影响）。执行攻击，按领域专长而非仅按安全背景编组。对照现实世界的危害概率验证后果，而不只看技术是否成功。把发现送进“破坏—修复”闭环，让缓解措施被度量，而不只是被声明。要收敛需要多轮，单轮会留下可观的残余漏洞。[^2]

这条边界在实践中是模糊的，但写下来很容易。自动化用于：每一个 OWASP 类别、编码类探测、回归检测、基线扫描。人工用于：新的攻击假设、CBRN 与医疗内容、高风险产品的多语言覆盖、聊天机器人里的心理社会伤害，以及任何需要领域专长才能判断输出的地方。总成本比纯靠工具高得多，而这正是对的。

## 把发现记录成能被修掉的形式

一条没有严重级别、没有复现步骤、没有修复路径的发现，只是一句抱怨。最小字段集：

- **唯一 ID**，前缀是工具名加 OWASP 或 ATLAS 编号（`promptfoo-LLM01-2026-014`）。
- **攻击技术**，写成 ATLAS 的 AML.T 标识符。
- **复现：** 精确的提示词序列与模型响应，能直接粘进测试里。
- **严重级别：** Critical / High / Medium / Low，附理由。
- **受影响组件：** 模型、系统提示词、输出处理器、插件、检索源。
- **修复路径：** 模型层（微调）、提示词层（加固系统提示词）、架构层（输出过滤、信任隔离、下线某个工具）。
- **验证标准：** 证明修复堵住了这条具体发现的证据，*以及*一个改写过措辞的变体。

失败最多的就是验证这一步。把 garak 标出的那条原始 payload 修掉，重跑时那条探测通过了，可同一攻击换个说法照样生效。garak 的论文在固定测试集的语境下把这种情形叫做 “dataset rot”；微软把它写成“破坏—修复”闭环必须度量缓解效果、而不能只重跑原探测的原因。[^2] 要跟踪*探测类别*的失败率，而不是单条探测的失败率。只有当同一攻击至少三种语义变体都通过时，才把一个类别标记为关闭。

这些修复路径与你在本部分前面已经建好的防御是对齐的：来自 [Guardrails](./02-guardrails.md) 的输入过滤与输出校验；来自 [Agent security and the lethal trifecta](./01-agent-security-and-the-lethal-trifecta.md) 的架构隔离；来自 [Compliance and audit](./05-compliance-and-audit.md) 的审计记录，用来证明修复真的上线了。微软对 Skeleton Key 自己的缓解手段是 Azure AI Content Safety 里的 Prompt Shields，加上对 Copilot 输出过滤与滥用监控的更新；模型本身没有重新训练。[^1] 靠的是分层防御，不是英雄式的对齐工程。

## 下一次渗透测试之前该上线什么

如果你的 AI 功能已经在生产环境跑，而还没有红队测试项目，第一周要做的并不多：

- 把 promptfoo 接进 CI，用一组第 1 层插件，覆盖 `prompt-injection`、`pii:direct`、`rbac`、`bola`、`excessive-agency`。配好 GitHub Actions，让它在相对 main 分支基线出现回归时失败。
- 在合入时安排一次第 2 层 garak 扫描，探测族对准你应用实际暴露的攻击面。
- 建好发现模板，以及一个 Jira 或 Linear 队列，带 ATLAS AML.T 标识符和 OWASP 类别标签。
- 挑一个第 3 层候选项（用 PyRIT 的 Crescendo 打你流量最大的那个 agent），每月手工跑一次，直到能改成每晚自动跑。

“我们做过安全训练”和“我们的系统是安全的”之间的那道缝，正是红队测试要补上的东西。截至 2024 年中，每一个前沿模型都倒在了 Skeleton Key 面前。截至 2026 年 6 月，还没有哪个公开的防御框架能消掉提示词注入。把那个能在别人之前发现自己系统失败的机制建起来，就是这件事本身。

## 参考文献

[^1]: Mark Russinovich, "Mitigating Skeleton Key, a new type of generative AI jailbreak technique," Microsoft Security Blog, 26 June 2024, https://www.microsoft.com/en-us/security/blog/2024/06/26/mitigating-skeleton-key-a-new-type-of-generative-ai-jailbreak-technique/

[^2]: Blake Bullwinkel and Ram Shankar Siva Kumar, "3 takeaways from red teaming 100 generative AI products," Microsoft Security Blog, 13 January 2025, https://www.microsoft.com/en-us/security/blog/2025/01/13/3-takeaways-from-red-teaming-100-generative-ai-products/

[^3]: garak, PyPI package v0.10.3, NVIDIA, March 2025, https://github.com/NVIDIA/garak

[^4]: Leon Derczynski et al., "garak: A Framework for Security Probing Large Language Models," arXiv:2406.11036, December 2024, https://arxiv.org/abs/2406.11036

[^5]: Microsoft Security Blog, "Announcing Microsoft's open automation framework to red team generative AI systems," 22 February 2024, https://www.microsoft.com/en-us/security/blog/2024/02/22/announcing-microsofts-open-automation-framework-to-red-team-generative-ai-systems/

[^6]: PyRIT Documentation v0.14.0, Microsoft, June 2026, https://microsoft.github.io/PyRIT/0.14.0/

[^7]: arXiv:2507.02956, "A Representation Engineering Perspective on the Effectiveness of Multi-Turn Jailbreaks," July 2025, https://arxiv.org/abs/2507.02956

[^8]: promptfoo, "Red Team Plugins," promptfoo.dev, last updated 12 June 2026, https://www.promptfoo.dev/docs/red-team/plugins/

[^9]: Microsoft Agent Governance Toolkit, "OWASP LLM Top 10 Mapping," 2025, https://microsoft.github.io/agent-governance-toolkit/compliance/owasp-llm-top10-mapping/

[^10]: OWASP GenAI Project, "OWASP Top 10 for LLM Applications 2025," 17 November 2024, https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/

[^11]: Repello AI, "MITRE ATLAS Framework: AI Attack Techniques (AML.T) Mapped to Red-Team Operations," 27 March 2026, https://repello.ai/blog/mitre-atlas-framework

[^12]: garak SDK, PyPI package `garak-sdk`, May 2026, https://pypi.org/project/garak-sdk/

[^13]: GoDaddy Engineering, "Beyond SAST: Automating AI Agent Security with Nemesis," May 2026, https://www.godaddy.com/resources/news/beyond-sast-automating-ai-agent-security-with-nemesis
