---
title: "变更上线"
description: "提示词与模型变更都是部署。以评测为门禁的晋级、影子流量、金丝雀发布、即时回滚，这套纪律把随手改个提示词变成安全生产变更。"
slug: 05-shipping-changes
part: 10
chapter: 5
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
  - part-3-prompt-engineering/04-prompt-versioning
  - part-8-evaluation-and-observability/04-regression-suites-and-ci-gates
  - part-8-evaluation-and-observability/07-monitoring-and-dashboards
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 变更上线

2025 年 4 月 25 日，OpenAI 为 GPT-4o 上线了一次针对默认人格的 system prompt 调整。错误率没有变化。延迟正常。每一块基础设施看板都是绿的。模型已经悄悄变得“过度奉承、过度附和”，谄媚到不管准不准都告诉用户他们想听的话。[^1] 出问题的信号来自社交媒体截图，不是内部监控。

回滚花了四天，因为它需要一次新的模型更新，而不是简单的版本切换。等回滚完成时，五亿周活用户已经全部暴露过了。[^1] mlops.community 的事后分析说得很直白：“一次金丝雀发布本可以把爆炸半径限制在一小撮人身上，而不是 1.8 亿月活用户。社交媒体上的骂声实际上成了主要的告警系统。”[^2]

这就是本章的核心主张：改提示词是一次部署，升模型版本也是。两者都该用你对待微服务发布的那套纪律。麻烦在于，LLM 的回归会带着自然语言的垃圾内容返回 HTTP 200，而不是 5xx 错误，这意味着你所有关于传统部署监控的知识都是必要的，但不够。

## 为什么“在线上直接改提示词”没有好下场

在多数团队里，提示词一开始只是嵌在应用代码里的字符串，或者直接粘在厂商的控制台里。在生产环境直接编辑它们，会绕过每一层安全防护：git 里没有版本、没有可评审的 diff、没跑过评测、没有审计记录。失败路径是机械式的：

1. 工程师在控制台里改提示词。
2. 变更立刻对 100% 流量生效。
3. 输出分布发生漂移，而且只在冷门输入上才显形。
4. 几天后，一张支持工单把问题捅出来。
5. 回滚意味着要从截图和 Slack 聊天记录里反推之前的行为。

标准的基础设施监控在结构上就对行为回归是瞎的。一个把工单路由到错误团队、生成法律上不正确的免责声明、或者滑进谄媚语气的 LLM，照样以正常延迟返回 200 OK。唯一的信号在下游：一张客诉工单、一个点踩、某个业务指标上的异常。[^3]

解法是把提示词（连同解码参数、模型版本锁定）当成一等公民的产物：有版本、可评审、有门禁、可回滚。到 2026 年，做这件事的工具链已经成熟。摩擦是可以接受的，只要你把框架摆对：这套纪律是让团队放手快跑的安全网，而不是迭代的刹车。

## 以评测为门禁的晋级

任何提示词变更要过的第一道门，是针对一份精选黄金数据集（golden dataset）的自动评测。如果通过率跌破阈值，CI 检查就失败，合并被挡住。

一个能用的门禁长什么样，参考 promptfoo 的 CI/CD 文档：[^4]

- **路径过滤。** 只有 `prompts/`、`promptfooconfig.yaml` 或模型配置文件被改动时，评测 CI 任务才触发。别在无关的 PR 上跑一次 3 分钟的评测。
- **黄金数据集进版本库。** 与提示词一起做版本控制。想要任何统计效力，至少 50 条；快速 CI 档位用 100-200 条。要包含有代表性的边界样例，不能只有简单输入。
- **质量门禁条件。** 一个脚本读评测输出、算通过率，低于阈值就以非零状态退出（快速档常用 95%）。
- **两层套件。** 快速档（50-200 条，3 分钟内，1 美元以内）跑在每个 PR 上。全量套件（500-2,000 条）每晚或发布分支上跑。

```python
import json, sys

def check_quality_gate(results_path: str, threshold: float = 0.95) -> None:
    with open(results_path) as f:
        data = json.load(f)
    stats = data["results"]["stats"]
    total = stats["successes"] + stats["failures"]
    if total == 0:
        sys.exit("No eval results; failing gate.")
    pass_rate = stats["successes"] / total
    print(f"Pass rate: {pass_rate:.1%} (threshold: {threshold:.1%})")
    if pass_rate < threshold:
        sys.exit(f"GATE FAILED: {pass_rate:.1%} < {threshold:.1%}")
    print("Gate passed.")
```

阈值标定的坑是真实存在的。门禁设得太紧（50 条样本上要 99%），LLM 的不确定性会让它无故触发。研究记录过，在 temperature 0 下、输入完全相同，由于 GPU 浮点运算差异，多次运行之间的准确率波动可达 15%。[^5] 一个会抖的门禁，会训练团队学会忽略它。设得太松（80%）又会漏掉真实回归。中间路线是：用双侧阈值（当前运行低于滚动基线 2 个标准差以上就告警），并且在把任何 LLM-as-judge 评分器提升为阻塞门禁之前，先按实测的人工评审一致率做标定。

至于评测集本身怎么构造，见[你的第一个评测集](../part-4-eval-fundamentals/02-your-first-eval-set.md)。CI 门禁机制深入讲，见[回归套件与 CI 门禁](../part-8-evaluation-and-observability/04-regression-suites-and-ci-gates.md)。本章负责的是套在那道门禁外面的部署纪律。

## 影子流量（shadow traffic）

评测门禁过了之后，重大变更的下一层是影子模式（shadow mode）：把每一个线上生产请求同时发给生产提示词和候选提示词，并行跑。候选的回复只记日志、永不下发。用户照常拿到生产侧的回复。

影子模式让你在零用户风险的前提下，拿到生产规模、真实生产输入下的行为数据。麻烦在评测这一层：一堆记下来的影子输出，只有配上自动比对才有用。标准做法是让 LLM 评审按质量标准（任务完成度、格式合规、事实准确性、语气）给两份回复打分，同时记录 token 数、成本和延迟的差异。

```python
import asyncio, logging, time

logger = logging.getLogger(__name__)

async def shadow_route(user_input: str, prod_system: str, candidate_system: str) -> str:
    """Send to prod and candidate in parallel. Return only the prod response.
    Log candidate output for offline judge comparison.
    """
    async def call(system: str) -> tuple[str, float]:
        start = time.monotonic()
        output = await llm_call(user_input, system)  # 具体实现相关
        return output, (time.monotonic() - start) * 1000

    (prod_out, prod_lat), (cand_out, cand_lat) = await asyncio.gather(
        call(prod_system), call(candidate_system)
    )
    logger.info("shadow_result", extra={
        "prod_output": prod_out, "candidate_output": cand_out,
        "prod_latency_ms": prod_lat, "candidate_latency_ms": cand_lat,
    })
    return prod_out  # 用户永远看到生产版本
```

代价是影子窗口内推理开销大约翻倍，因为每个请求你都在跑两个模型。这就是零用户暴露要付的账。

影子模式对重大变更才划算：模型版本升级、提示词大幅重构、新的工具 schema、安全护栏调整。小改动用评测门禁就够了，不值得为它花这份钱。Ramp 在为其费用 agent 上线每个新提示词或模型版本之前，都会先在影子模式下跑真实财务交易；LLM 评审把预测动作和人的实际决策作比对，只有影子准确率过了阈值，才放开真实动作。[^6]

更便宜的变体是离线影子模式：把上周的生产 trace 回放到候选版本上，离线跑评审。mlops.community 对 GPT-4o 事件的分析结论是，对历史流量做离线影子“很可能在用户暴露之前就显现出过度吹捧的分布漂移”。[^2] 基础设施比实时影子少，但你拿不到回放窗口之后才出现的输入。

## 金丝雀发布（canary rollout）

金丝雀把一小部分真实用户流量路由到候选版本，其余留在生产。标准爬坡：从 1% 起（高风险应用有时从 0.1% 起），停住观察，信号干净就依次升到 5%、20%、50%、100%。任何指标超阈值就立刻回滚。

> [!WARNING]
> **用户粘性分配是强制的。** 按请求随机分配会让用户体验不连贯：同一个用户在一段对话里看到的回复风格都不一样。用用户 ID 的稳定哈希，让每个用户确定性地映射到某个分组。

```python
import hashlib
from dataclasses import dataclass, field

@dataclass
class CanaryConfig:
    canary_pct: float = 0.05
    prod_version: str = "v1"
    candidate_version: str = "v2"
    rollback_threshold: float = 0.10  # 错误率差值
    metrics: dict[str, list[float]] = field(default_factory=dict)

    def record(self, version: str, success: bool) -> None:
        self.metrics.setdefault(version, []).append(1.0 if success else 0.0)

    def error_rate(self, version: str) -> float:
        vals = self.metrics.get(version, [])
        return 1.0 - sum(vals) / len(vals) if vals else 0.0

    def should_rollback(self) -> bool:
        return (self.error_rate(self.candidate_version)
                - self.error_rate(self.prod_version) > self.rollback_threshold)

def route_request(user_id: str, cfg: CanaryConfig) -> str:
    if cfg.should_rollback():
        return cfg.prod_version
    bucket = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 1000 / 1000
    return cfg.candidate_version if bucket < cfg.canary_pct else cfg.prod_version
```

LLM 金丝雀期间要看的信号，和普通服务发布不一样：

- **延迟分位数**（p50、p95、p99），不是平均值。LLM 的延迟是右偏的。
- **每请求成本。** token 数会随模型版本变化；等到 100% 流量才发现就贵了。
- **拒答率与错误率。** 新模型可能拒掉更多类别的请求。
- **输出长度分布。** 模式坍塌（回复极短）或失控的啰嗦，都是分布漂移的标志。
- **用户反馈信号。** 点踩率、重新生成请求、会话放弃，全部按分组统计。

自动回滚不是可选项。把阈值写明确：p99 延迟上升超过 40%、拒答率跳升超过 5%，或者每请求成本超预算，就在无人介入的情况下把 100% 流量切回生产。不该需要有人在凌晨两点被叫醒来修一个行为回归。

LaunchDarkly AI Configs（2024 年 6 月 13 日正式可用）把这件事变成运行时配置变更，而不是重新部署。[^7] 模型 ID、system prompt、解码参数都是 JSON flag 的变体；一条百分比放量规则控制分组成员；Release Guardian 自动发起回滚。Portkey 在网关层支持同样的模式。

## 即时回滚（rollback）

回滚是部署的前置条件，不是事发后的补救动作。Anthropic 的 Managed Agents cookbook 给出了这个模型：每次 `agents.update()` 都产出一个不可变版本号；生产调用方锁定到具体版本，写作 `agent={"type": "agent", "id": AGENT_ID, "version": 1}`；回滚就是把生产配置重新指向上一个版本 ID。不用重新部署。不用改代码。[^8]

cookbook 里的规则是：“生产调用方永远锁定显式版本，而不是裸的 agent ID。新版本在你晋级之前一直不可见。”[^8]

裸 agent ID 就是新的“在控制台里改提示词”。同一个反模式，只是通过一次 API 调用自动化了。如果你的应用代码传的是裸 ID，任何有工作区权限的人一次 `agents.update()` 调用就立刻影响所有运行中的会话，没有任何评审门禁。

标准回滚指标漏掉的东西：HTTP 错误率、p99 延迟、异常计数都测不出行为回归。谄媚的 GPT-4o 以正常延迟返回 200 OK。回滚信号必须来自行为指标：按分组的 LLM 评审分数、点踩率、拒答率、输出长度分布。这就是为什么金丝雀要排在回滚之前而不是被回滚替代：没有金丝雀分组去发现回归，等你知道的时候，100% 的用户已经中招了。

回滚就绪是部署检查清单上的一项：

- 上一个版本 ID 记录在 PR 描述里。
- 存在把生产重新指向上一个版本的 runbook，并且在金丝雀放量之前已在 staging 演练过。
- 行为指标已配置好，能发现这次回归。

## 完整流水线

拼起来看，一次提示词或模型变更的流水线是这样：

1. **带提示词 diff 的 PR** 触发快速评测 CI 门禁（路径过滤、50-200 条样本、3 分钟内、95% 阈值）。
2. **PR 评审**，评测结果 diff 作为评论贴出来。
3. **合并**把提示词晋级为一个新的不可变版本。
4. **影子模式**用于重大变更（模型版本升级、提示词大改、schema 变更）。
5. **1% 金丝雀**，带用户粘性分配和行为指标看板。
6. **爬到 100%**，前提是信号一直干净，全程保持自动回滚阈值生效。
7. **每一步都可回滚**：上一个版本 ID 已知，runbook 演练过。

值得认真对待的反对意见来自 Hamel Husain（他给 2,000 多名工程师讲过 AI 评测）：不是每个变更都需要全套纪律。内部效率工具可能不需要金丝雀；面向客户的财务 agent 绝对需要。过度工程化的代价是迭代变慢；工程不足的代价是一次生产事故。按风险大小标定，但每一个变更至少需要版本化的提示词和经过验证的回滚方案。[^9]

除了显而易见的“在控制台里改提示词”，还有两个反模式要盯：

- **把 SDK 版本升级当成静默的模型发布。** 把 `openai` 从 1.x.y 升到 1.x.z 而不跑评测，可能就顺带发布了新的默认 system prompt 或 token 预算。评测门禁的路径过滤 `prompts/**` 覆盖不到 `pyproject.toml`。把依赖清单加进路径过滤，并给评测运行打上 SDK 版本标签。[^10]
- **奖励信号短视。** 评测套件通过了，金丝雀指标看着也正常，但模型已经被优化成追求短期点赞而不是长期满意度。GPT-4o 就是这么翻车的。把纵向指标（多会话留存、下游任务完成）放进评测设计里。

这套纪律一旦接好并不贵。评测门禁是一个 CI workflow 文件。影子路由是一个 async 辅助函数。金丝雀分配是一个哈希函数。而*什么都不做*的代价，大致是四天的回滚暴露期，摊在五亿用户面前。这笔账没什么可算的。

## 参考文献

[^1]: OpenAI, "Sycophancy in GPT-4o: what happened and what we're doing about it", April 29, 2025, https://openai.com/index/sycophancy-in-gpt-4o/
[^2]: Han Lee, "When Prompt Deployment Goes Wrong: MLOps Lessons from ChatGPT's Sycophantic Rollback", mlops.community, May 6, 2025, https://home.mlops.community/en/public/blogs/when-prompt-deployment-goes-wrong-mlops-lessons-from-chatgpts-sycophantic-rollback
[^3]: Tian Pan, "Releasing AI Features Without Breaking Production: Shadow Mode, Canary Deployments, and A/B Testing for LLMs", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-09-llm-gradual-rollout-shadow-canary-ab-testing
[^4]: promptfoo, "CI/CD Integration for LLM Evaluation and Security", official documentation, June 2026, https://www.promptfoo.dev/docs/integrations/ci-cd/
[^5]: Tian Pan, "LLM Non-Determinism in Production Evaluations", TianPan.co, 2026 (cited in [^3])
[^6]: Alex Strick van Linschoten, "What 1,200 Production Deployments Reveal About LLMOps in 2025", ZenML, December 2025, https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025
[^7]: LaunchDarkly, "Introducing AI Model and AI Prompt Flags (GA)", June 13, 2024, https://launchdarkly.com/blog/introducing-ai-model--ai-prompt-flags/
[^8]: Anthropic, "Managed Agents tutorial: prompt versioning and rollback", Anthropic Cookbook, April 2026, https://platform.claude.com/cookbook/managed-agents-cma-prompt-versioning-and-rollback
[^9]: Hamel Husain, "AI Evals FAQ", hamel.dev, 2026, https://hamel.dev/blog/posts/evals-faq/
[^10]: Tian Pan, "The SDK Upgrade Tax: When a Dependency Bump Becomes a Model Deployment", TianPan.co, April 2026, https://tianpan.co/blog/2026-04-28-llm-sdk-upgrade-tax-prompt-behavior-eval-gate
