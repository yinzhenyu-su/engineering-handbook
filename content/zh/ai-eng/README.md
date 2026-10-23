# AI Engineering Handbook（AI 工程手册）

**作为软件工程师，如何构建并运营生产级 AI 产品。**

12 个部分共 86 章 · 约 192,000 词 · 87 张插图 · 5 个 Mermaid 图 · 200 个 Python 示例 · 757 条带脚注的引用

可以在 GitHub 上直接读，也可以访问网站 [ai-eng.handbook.academy](https://ai-eng.handbook.academy)。

---

## 这本书讲什么

AI 工程师在基础模型之上做产品，他们不训练模型。这本书把界线划在模型 API 上，然后就待在 API 的产品这一侧：模型如何表现，如何可靠地调用，如何拼装模型看到的上下文，如何做检索，如何搭建 agent，如何评测和观测这一切，如何做安全，以及如何把成本压到业务能接受的水平来运行它。

这不是一门机器学习课程。分词、注意力、采样只在它们变成你能实际拧动的成本、延迟或质量旋钮时才出现。从零训练和架构研究不在范围内。微调只有一个决策章节，[定制化菜单](part-10-deployment-and-cost/02-the-customization-menu.md)，因为大多数团队应该先把更便宜的每一级台阶爬完。

## 与 HLD Handbook 的关系

[HLD 第 9 部分：AI 与 ML 系统设计](../hld/part-9-ai-ml-system-design/)是白板上的事：如何在规模上设计一个 AI 系统。这本书是编辑器：如何把一个 AI 系统建起来并跑起来。当架构尺度的视角有帮助时，这里的章节会链到对应的 HLD 章节。

## 怎么读这本书

第一遍按顺序读。这个顺序是刻意排的。

- **第 0 部分**建立心智模型。[升级阶梯](part-0-start-here/03-the-escalation-ladder.md)（提示词、上下文、检索、工具、agent、微调）是全书反复回到的那条决策规则。
- **第 1 至 3 部分**讲原材料：模型行为、API 和提示词。
- **第 4 部分刻意排在 RAG 和 agent 之前。** 后面每一部分都假定你手里已有一套评估集。如果只读一部分，就读这一部分。
- **第 5 至 7 部分**搭系统：上下文、检索、agent。
- **第 8 至 11 部分**把它跑起来：更深入的评测、可观测性、安全、生产运维和产品。

每章的结构取决于主题需要什么。多数章节从一个真实事故或一个失败的例子开场，讲清机制，最后落在一个能直接用的东西上：一份检查清单、一条决策规则，或一段能跑起来的代码。

## 约定

- **代码用 Python。** 示例面向 Python 3.10+ 和各家官方 SDK（openai、anthropic、pydantic、httpx）。
- **会过时的事实都带日期。** 价格、模型名、上下文上限、基准分数都变得很快。规矩是每一条都标上“截至”日期和带脚注的来源。数字过期了，就带上新来源提一个 PR。
- **图放在章节旁边**，在各部分的 `images/` 目录里，配描述性 alt 文本和一行图注。只有当流程图比插图讲得更清楚时才用 Mermaid。
- **警告**用 GitHub 的 `> [!WARNING]` 提示块，专门标注那些让团队真金白银损失或出事故的生产陷阱。
- **Frontmatter** 遵循 `scripts/check-frontmatter.mjs` 校验的 schema：`slug` 与文件名一致，`part` 和 `chapter` 与路径一致，`prerequisites` 指向本书的其他章节。

## 课程目录

### 第 0 部分：从这里开始（4 章）

AI 工程师是什么、软件基线、在概率系统中思考，以及全书遵循的那条升级阶梯：提示词、上下文、检索、工具、agent、微调。

| # | 章节 | 级别 |
|---|---|---|
| 0.0 | [AI 工程师是什么（以及不是什么）](part-0-start-here/00-what-an-ai-engineer-is.md) | 入门 |
| 0.1 | [软件基线](part-0-start-here/01-the-software-baseline.md) | 入门 |
| 0.2 | [在概率系统中思考](part-0-start-here/02-thinking-in-probabilistic-systems.md) | 入门 |
| 0.3 | [升级阶梯](part-0-start-here/03-the-escalation-ladder.md) | 入门 |

### 第 1 部分：模型如何表现（7 章）

模型全景、token 与上下文窗口、采样、嵌入向量、推理模型、成本-质量-延迟三角，以及刚好够用来做系统决策的机器学习。

| # | 章节 | 级别 |
|---|---|---|
| 1.0 | [模型全景图](part-1-how-models-behave/00-the-model-landscape.md) | 入门 |
| 1.1 | [Token 与上下文窗口](part-1-how-models-behave/01-tokens-and-context-windows.md) | 入门 |
| 1.2 | [采样与 temperature](part-1-how-models-behave/02-sampling-and-temperature.md) | 入门 |
| 1.3 | [嵌入向量（Embeddings）](part-1-how-models-behave/03-embeddings.md) | 入门 |
| 1.4 | [推理模型](part-1-how-models-behave/04-reasoning-models.md) | 入门 |
| 1.5 | [成本-质量-延迟三角](part-1-how-models-behave/05-the-cost-quality-latency-triangle.md) | 中级 |
| 1.6 | [够用的机器学习](part-1-how-models-behave/06-just-enough-ml.md) | 入门 |

### 第 2 部分：使用模型 API（8 章）

可靠地调用模型：流式输出、结构化输出、工具调用、错误与回退、模型选择、网关与路由，以及提示词缓存的经济账。

| # | 章节 | 级别 |
|---|---|---|
| 2.0 | [调用模型](part-2-working-with-model-apis/00-calling-models.md) | 入门 |
| 2.1 | [流式输出](part-2-working-with-model-apis/01-streaming.md) | 入门 |
| 2.2 | [结构化输出](part-2-working-with-model-apis/02-structured-outputs.md) | 入门 |
| 2.3 | [函数调用与工具调用](part-2-working-with-model-apis/03-function-and-tool-calling.md) | 中级 |
| 2.4 | [错误、重试与回退](part-2-working-with-model-apis/04-errors-retries-fallbacks.md) | 中级 |
| 2.5 | [如何选模型与读懂基准测试](part-2-working-with-model-apis/05-choosing-models-and-reading-benchmarks.md) | 入门 |
| 2.6 | [网关与路由](part-2-working-with-model-apis/06-gateways-and-routing.md) | 中级 |
| 2.7 | [提示词缓存](part-2-working-with-model-apis/07-prompt-caching.md) | 入门 |

### 第 3 部分：提示词工程（5 章）

生产级提示词：解剖、少样本、思维链与推理模型带来的反转、模板与链式调用，以及可回滚的提示词版本管理。

| # | 章节 | 级别 |
|---|---|---|
| 3.0 | [生产级提示词的解剖](part-3-prompt-engineering/00-anatomy-of-a-production-prompt.md) | 入门 |
| 3.1 | [零样本与少样本](part-3-prompt-engineering/01-zero-shot-and-few-shot.md) | 入门 |
| 3.2 | [思维链、ReAct 与自一致性](part-3-prompt-engineering/02-chain-of-thought-react-self-consistency.md) | 中级 |
| 3.3 | [模板与链式调用](part-3-prompt-engineering/03-templates-and-chaining.md) | 中级 |
| 3.4 | [提示词版本管理](part-3-prompt-engineering/04-prompt-versioning.md) | 中级 |

### 第 4 部分：评测基础（4 章）

最小评测闭环，刻意排在 RAG 和 agent 之前：先看你的数据，建起第一套 50 条示例的评估集，把断言接进 CI。这一部分之后的任何东西，没有评测都不上线。

| # | 章节 | 级别 |
|---|---|---|
| 4.0 | [为什么没有评测就上不了线](part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md) | 入门 |
| 4.1 | [先看你的数据](part-4-eval-fundamentals/01-look-at-your-data.md) | 入门 |
| 4.2 | [你的第一套评估集](part-4-eval-fundamentals/02-your-first-eval-set.md) | 入门 |
| 4.3 | [LLM 输出的断言与单元测试](part-4-eval-fundamentals/03-assertions-and-unit-tests.md) | 入门 |

### 第 5 部分：上下文工程（6 章）

构建模型看到的完整世界：上下文组装与路由、压缩与预算、记忆与状态、工具上下文，以及测试真正被组装出来的东西。

| # | 章节 | 级别 |
|---|---|---|
| 5.0 | [上下文不只是提示词](part-5-context-engineering/00-context-is-more-than-the-prompt.md) | 中级 |
| 5.1 | [上下文组装与路由](part-5-context-engineering/01-context-assembly-and-routing.md) | 中级 |
| 5.2 | [压缩与上下文预算](part-5-context-engineering/02-compression-and-context-budgets.md) | 中级 |
| 5.3 | [记忆与状态](part-5-context-engineering/03-memory-and-state.md) | 中级 |
| 5.4 | [工具上下文与元数据](part-5-context-engineering/04-tool-context-and-metadata.md) | 中级 |
| 5.5 | [测试与观测上下文](part-5-context-engineering/05-testing-and-observing-context.md) | 中级 |

### 第 6 部分：检索（RAG）（9 章）

把检索做对：什么时候真的需要 RAG、分块、向量检索与混合检索、重排序、查询理解、引用、检索评估，以及生产环境的索引构建。

| # | 章节 | 级别 |
|---|---|---|
| 6.0 | [到底什么时候才真的需要 RAG](part-6-retrieval-augmented-generation/00-when-you-actually-need-rag.md) | 中级 |
| 6.1 | [文档处理与分块](part-6-retrieval-augmented-generation/01-document-processing-and-chunking.md) | 中级 |
| 6.2 | [向量嵌入与向量检索](part-6-retrieval-augmented-generation/02-embeddings-and-vector-search.md) | 中级 |
| 6.3 | [混合检索与重排序](part-6-retrieval-augmented-generation/03-hybrid-search-and-reranking.md) | 中级 |
| 6.4 | [查询理解](part-6-retrieval-augmented-generation/04-query-understanding.md) | 中级 |
| 6.5 | [引用与依据](part-6-retrieval-augmented-generation/05-citations-and-grounding.md) | 中级 |
| 6.6 | [高级检索](part-6-retrieval-augmented-generation/06-advanced-retrieval.md) | 进阶 |
| 6.7 | [评估检索](part-6-retrieval-augmented-generation/07-evaluating-retrieval.md) | 中级 |
| 6.8 | [生产级 RAG](part-6-retrieval-augmented-generation/08-production-rag.md) | 进阶 |

### 第 7 部分：Agent 工程（13 章）

workflow 与 agent 之别、工具设计、MCP 及其安全模型、记忆、人在回路、长时间运行的智能体、多智能体系统、agent 原型、语音智能体，以及如何选框架。

| # | 章节 | 级别 |
|---|---|---|
| 7.0 | [Workflow 与 agent](part-7-agent-engineering/00-workflow-vs-agent.md) | 中级 |
| 7.1 | [五种工作流模式](part-7-agent-engineering/01-the-five-workflow-patterns.md) | 中级 |
| 7.2 | [Agent 循环](part-7-agent-engineering/02-the-agent-loop.md) | 中级 |
| 7.3 | [工具设计](part-7-agent-engineering/03-tool-design.md) | 中级 |
| 7.4 | [MCP：模型上下文协议](part-7-agent-engineering/04-mcp-model-context-protocol.md) | 中级 |
| 7.5 | [MCP 安全](part-7-agent-engineering/05-mcp-security.md) | 中级 |
| 7.6 | [智能体记忆与任务状态](part-7-agent-engineering/06-agent-memory-and-task-state.md) | 中级 |
| 7.7 | [人在回路（Human-in-the-loop）](part-7-agent-engineering/07-human-in-the-loop.md) | 中级 |
| 7.8 | [长时间运行的智能体](part-7-agent-engineering/08-long-running-agents.md) | 进阶 |
| 7.9 | [多智能体系统](part-7-agent-engineering/09-multi-agent-systems.md) | 进阶 |
| 7.10 | [Agent 原型](part-7-agent-engineering/10-agent-archetypes.md) | 中级 |
| 7.11 | [语音智能体](part-7-agent-engineering/11-voice-agents.md) | 中级 |
| 7.12 | [选择框架](part-7-agent-engineering/12-choosing-a-framework.md) | 中级 |

### 第 8 部分：评测与可观测性（9 章）

全书的主心骨：LLM-as-judge 及其失败模式、面向 RAG 与 agent 的分组件评测、CI 里的回归门禁、在线评估、链路追踪、监控，以及数据飞轮。

| # | 章节 | 级别 |
|---|---|---|
| 8.0 | [评测分类法](part-8-evaluation-and-observability/00-the-eval-taxonomy.md) | 中级 |
| 8.1 | [LLM-as-judge](part-8-evaluation-and-observability/01-llm-as-judge.md) | 中级 |
| 8.2 | [裁判的失败模式](part-8-evaluation-and-observability/02-judge-failure-modes.md) | 中级 |
| 8.3 | [评估 RAG 与 agent](part-8-evaluation-and-observability/03-evaluating-rag-and-agents.md) | 中级 |
| 8.4 | [回归套件与 CI 门禁](part-8-evaluation-and-observability/04-regression-suites-and-ci-gates.md) | 中级 |
| 8.5 | [在线评估与 A/B 测试](part-8-evaluation-and-observability/05-online-evaluation-and-ab-testing.md) | 中级 |
| 8.6 | [链路追踪](part-8-evaluation-and-observability/06-tracing.md) | 中级 |
| 8.7 | [监控与仪表盘](part-8-evaluation-and-observability/07-monitoring-and-dashboards.md) | 中级 |
| 8.8 | [反馈与数据飞轮](part-8-evaluation-and-observability/08-feedback-and-the-data-flywheel.md) | 中级 |

### 第 9 部分：安全与治理（7 章）

提示词注入、致命三要素与 agent 安全、护栏、防滥用、PII 与隐私、合规，以及给自己的系统做红队测试。

| # | 章节 | 级别 |
|---|---|---|
| 9.0 | [提示词注入](part-9-security-and-governance/00-prompt-injection.md) | 中级 |
| 9.1 | [Agent 安全与致命三要素](part-9-security-and-governance/01-agent-security-and-the-lethal-trifecta.md) | 中级 |
| 9.2 | [护栏](part-9-security-and-governance/02-guardrails.md) | 中级 |
| 9.3 | [防滥用](part-9-security-and-governance/03-abuse-prevention.md) | 中级 |
| 9.4 | [PII 与隐私](part-9-security-and-governance/04-pii-and-privacy.md) | 进阶 |
| 9.5 | [合规与审计](part-9-security-and-governance/05-compliance-and-audit.md) | 进阶 |
| 9.6 | [红队测试](part-9-security-and-governance/06-red-teaming.md) | 进阶 |

### 第 10 部分：生产环境中的 AI（8 章）

在生产环境里运营 AI：托管与自托管之别、定制化菜单、成本工程与延迟工程、以评测为门禁的发布、SLO、事故响应，以及 AI 基础设施模式。

| # | 章节 | 级别 |
|---|---|---|
| 10.0 | [托管还是自托管](part-10-deployment-and-cost/00-hosted-vs-self-hosted.md) | 中级 |
| 10.1 | [自托管要点](part-10-deployment-and-cost/01-self-hosting-essentials.md) | 中级 |
| 10.2 | [定制化菜单](part-10-deployment-and-cost/02-the-customization-menu.md) | 中级 |
| 10.3 | [成本工程](part-10-deployment-and-cost/03-cost-engineering.md) | 中级 |
| 10.4 | [延迟工程](part-10-deployment-and-cost/04-latency-engineering.md) | 中级 |
| 10.5 | [变更上线](part-10-deployment-and-cost/05-shipping-changes.md) | 中级 |
| 10.6 | [SLO 与事故响应](part-10-deployment-and-cost/06-slos-and-incident-response.md) | 中级 |
| 10.7 | [AI 基础设施模式](part-10-deployment-and-cost/07-ai-infrastructure-patterns.md) | 中级 |

### 第 11 部分：AI 产品工程（6 章）

AI UX、失败恢复、准确率之外的产品指标、实验、成本建模与定价，以及会复利增长的反馈回路。

| # | 章节 | 级别 |
|---|---|---|
| 11.0 | [AI 产品的用户体验（UX）](part-11-ai-product/00-ai-ux.md) | 中级 |
| 11.1 | [失败恢复的用户体验](part-11-ai-product/01-failure-recovery-ux.md) | 中级 |
| 11.2 | [AI 产品指标](part-11-ai-product/02-ai-product-metrics.md) | 中级 |
| 11.3 | [实验](part-11-ai-product/03-experimentation.md) | 进阶 |
| 11.4 | [成本建模与定价](part-11-ai-product/04-cost-modeling-and-pricing.md) | 中级 |
| 11.5 | [反馈与增长回路](part-11-ai-product/05-feedback-and-growth-loops.md) | 进阶 |

## 参与贡献

纠错是最有价值的贡献：一个过期的价格、一个改名的 API 参数、一个已被取代的基准、一段已经跑不动当前 SDK 的代码。见 [CONTRIBUTING.md](../../../CONTRIBUTING.md) 和 [STYLE_GUIDE.md](../../../STYLE_GUIDE.md)。开 PR 之前先跑 `npm run check:all`。

## 许可协议

[CC BY-SA 4.0](../../LICENSE)，与本仓库中每一本手册相同。
