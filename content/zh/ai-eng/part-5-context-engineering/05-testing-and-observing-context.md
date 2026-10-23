---
title: "测试与观测上下文"
description: "把组装好的上下文做成快照，对组装逻辑做回归测试，并采集能回答“模型为什么看到这些内容”的 traces（追踪）。"
slug: 05-testing-and-observing-context
part: 5
chapter: 5
difficulty: intermediate
prerequisites:
  - 00-context-is-more-than-the-prompt
  - 01-context-assembly-and-routing
  - part-4-eval-fundamentals/02-your-first-eval-set
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 测试与观测上下文

一个团队在周二上线。CI 全绿。到了周三早上，客服开始收工单：助手不再标注引用来源了。有人翻出测试报告，发现上周的 PR 里“更新”了 47 个提示词快照，理由大多只有一行：*重新生成*。没人看过 diff。而引用来源那一行，早在六个提交之前就从系统消息里被悄悄删掉了。

这个故事就是你要防的那种故障模式。你的上下文，就是代码组装好、发给模型的那个 messages 数组：系统指令、检索到的文档、对话历史、工具 schema、用户这一轮输入。模型从不把它回显出来。你的日志多半也没存它。而一旦把它接进 CI 快照，最自然的冲动就是把快照当成模型回复一样对待——这就是 bug 本身。

解法是把上下文的测试放在两个完全分离的位置，用两种不同的断言，因为你的上下文横跨两个层次，而这两个层次的失败方式不一样。

## 两个层次，两套测试策略

第一层是你的**组装代码**：那个接收用户查询、检索片段、对话历史和提示词模板，返回一个 `messages` 数组的函数。输入相同，输出就相同。它就是普通的确定性 Python。你可以对它取哈希、做 diff、免费跑一千条断言，一次 API 调用都不用。

第二层是**模型的行为**，也就是你把这个数组发出去之后发生的事。它是随机的。同样的输入可以返回不同的输出，供应商会在你不知情的情况下打补丁，而“字符串完全相等”这条断言，对这条分界线另一侧的一切都是错的。

![一条竖直虚线把画布分成两半：左边是组装代码的工作区，包含属性测试、哈希快照和 token 预算检查，一起汇入绿色的 CI 横幅；右边是随机的模型层，包含一个黄金样本群、一道聚合得分门禁和 LLM-as-judge，汇入黄色的部署前横幅；中间一支标记为“messages[]”的箭头跨过边界](../../../ai-eng/part-5-context-engineering/images/05-two-layer-testing.png)
*用哈希和属性测试组装逻辑；用样本群门禁测试模型行为。分界线就是你的代码调用供应商的那一刻。*

搞错这件事的团队，会把两层塞进同一个测试里：一个快照既序列化组装好的 messages，*也*序列化模型的回复，两条都断言，然后每个 PR 都跑一遍。这种测试套件会产出被模型补丁搅乱的噪声 diff、因为真实 API 调用而变慢的 CI，以及一种“不读就重新生成快照”的团队习惯。几个月之内，它就什么都抓不到了。[^1] 把两层拆开，每个测试只干好一件事。

## 只快照组装好的 messages，别的都不要

你能写的最便宜、最有用的测试，就是给组装好的上下文取哈希。拿一组固定输入跑组装函数，把 messages 数组序列化，算出 SHA-256，把这个哈希提交进仓库。下一个 PR 来了就重算哈希；不一样就构建失败，等人来看 diff。

```python
import hashlib
import json

def snapshot_context(name: str, messages: list[dict]) -> dict:
    """Serialize and hash assembled context for snapshot storage."""
    serialized = json.dumps(messages, sort_keys=True, ensure_ascii=False)
    digest = hashlib.sha256(serialized.encode()).hexdigest()
    return {"name": name, "messages": messages, "sha256": digest}


def assert_snapshot(current: dict, stored: dict) -> None:
    """Fail if the assembled context changed without a deliberate update."""
    if current["sha256"] != stored["sha256"]:
        raise AssertionError(
            f"Context snapshot mismatch for '{current['name']}'. "
            "Run with --update-snapshots to accept the change."
        )
```

机制就这么点东西。Freeday 是一个每天多次部署的 AI 产品团队，用的就是这个模式：名字、messages、哈希，三个字段，CI 总共跑两分钟左右。这道哈希门禁决定要不要触发他们那套跑得很慢的 LLM 评测，所以大多数 PR 一个 token 都不花。[^2]

它能抓到的是那些悄无声息的东西。一次检索器重构把文档顺序换了。一次模板改动把系统消息弄丢了。一条新加的注入规则把用户内容悄悄塞到了指令前面。哈希翻了，diff 落在 PR 里，就必须有人去读。它抓不到的是上下文*对不对*：一个坏掉的组装逻辑，它生成的快照能永远通过。这不是缺陷，这是契约。快照的职责是发现变化，不是评判内容。

有一个错误要避免：别拿线上的检索器或线上 embedding API 来跑这个快照测试。CI 会因为网络错误抖动，测试数据会漂移，一个季度之内团队就会把这套测试关掉。把检索结果固定成仓库里的 JSON fixture，喂给组装函数。真实检索这条路，留给更慢的集成层。

## 属性测试能补上哈希漏掉的部分

哈希告诉你“有东西变了”。属性测试告诉你“不管输入是什么，哪些事永远应该成立”。给任何不是一眼看穿的组装函数写上一小把这样的测试，它们会比你的每一次提示词重写、每一次换模型、每一次检索器升级都活得久。

```python
def build_context(user_query: str, retrieved_docs: list[str]) -> list[dict]:
    system = "You are a helpful assistant. Answer using only the provided documents."
    context_block = "\n\n".join(f"[Doc {i+1}]: {d}" for i, d in enumerate(retrieved_docs))
    return [
        {"role": "system", "content": system},
        {"role": "user", "content": f"{context_block}\n\nQuestion: {user_query}"},
    ]


def test_system_message_first():
    ctx = build_context("query", ["doc1"])
    assert ctx[0]["role"] == "system", "System message must be first"


def test_all_docs_included():
    ctx = build_context("q", ["alpha", "beta", "gamma"])
    user_content = ctx[-1]["content"]
    for doc in ["alpha", "beta", "gamma"]:
        assert doc in user_content, f"Missing doc: {doc}"


def test_token_budget(tokenizer, max_tokens=8000):
    ctx = build_context("q" * 100, ["d" * 4000] * 5)
    total = sum(len(tokenizer.encode(m["content"])) for m in ctx)
    assert total < max_tokens, f"Context exceeds budget: {total}"
```

这三条断言把每个版本的 `build_context` 都应该遵守的不变量写死了。它们不怕改提示词，因为它们没有钉住具体措辞。它们不怕升级模型，因为它们根本不调模型。它们还能抓到哈希快照抓不到的那一类 bug：在测试输入上正确，换一个稍有不同的输入就挂了。几乎任何组装函数都值得写的一组断言：

- 系统消息存在，并且在第一位。
- 调用方传进来的每一篇文档，都在用户消息的某个位置出现过。
- 在最坏情况的输入下，总 token 数仍在模型的上下文上限之内。
- 角色全部合法（`system`、`user`、`assistant`、`tool`）。
- 没有连续两条消息属于同一个角色（多数供应商会拒绝这种请求）。

## 不要快照模型回复：用样本群做门禁

如果你已经把组装和行为拆开，问题就变成怎么测行为。错的答案是给模型回复做快照。供应商的模型一直在漂：悄悄打补丁、把别名指向新版本、下线旧版本。Ma 等人对 18 个月里五个 GPT-3.5 版本做了一项研究，发现 58.8% 的“提示词 + 模型”组合在至少一次更新之后准确率下降，其中 70.2% 的下降超过五个百分点。[^3] 即便整体准确率提高了，也有 10.9% 原本答对的预测退化了，而且常常发生在模型置信度最高的时候。`gpt-3.5-turbo` 的一次更新，让一种提示词设计的准确率掉了 9.6%，却让另一种提示词设计在完全相同的任务上*涨了* 5.1%。

回复快照就住在这个世界里。它会在噪声上失败，同时错过真正的信号。

对的工具是黄金样本群：50 到 300 条有代表性的输入，用你当前的组装逻辑加当前模型去评，把这批输入的聚合得分记成基线。你改组装逻辑时，重跑这批样本，用带容忍度的门禁比较聚合分（退化 2% 左右是常见的起点）。断言是统计性的，所以模型漂移不会让它来回打摆；样本群足够大，一两个古怪样本也不会把构建搞挂。

有两条运营规则能让样本群门禁长期有效。第一，每次跑样本群都把提示词版本和模型版本*一起*记录下来，因为同一次模型更新对不同提示词的影响不一样。第二，把样本群按类别切片（意图、语言、文档类型、用户分层），每一片单独设门禁，因为总体准确率可能纹丝不动，而某一片悄悄塌了 15%。切片要从第一天就内建进去；出了事故再补，很痛苦。

[你的第一套评测集](../part-4-eval-fundamentals/02-your-first-eval-set.md)讲的是样本群本身怎么搭。这里的活是把样本群接进流程：快照哈希决定这批样本到底要不要跑。如果组装出来的上下文和上次逐字节相同，样本群就没看到任何新东西，你可以省下这笔 API 开销。

## 生产环境里，记录模型真正看到的内容

测试证明的是罐头输入上的事。生产跑的是用户发来的任何东西，而凌晨三点告警响起时，唯一重要的问题是：*模型到底看到了什么？* 如果你的 traces（追踪）回答不了这个，你就是在猜。

OpenTelemetry 的 GenAI 语义约定（2026 年年中为 v1.41）恰恰定义的就是这件事。一个 `chat {model}` span 以结构化属性的形式携带系统指令、输入消息、工具定义和输出消息，同时还有厂商中立的用量计数器，在 OpenAI、Anthropic、Google 等各家都能用。[^4] 这个约定还处于“Development”状态，所以属性名仍可能变；把你的仪表盘钉到某个具体版本，并跟进 changelog。

```python
from opentelemetry import trace
import json

tracer = trace.get_tracer("my-app")

def trace_llm_call(messages: list[dict], model: str, capture_content: bool = False):
    with tracer.start_as_current_span(f"chat {model}") as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", "openai")
        span.set_attribute("gen_ai.request.model", model)
        if capture_content:
            span.set_attribute(
                "gen_ai.system_instructions",
                json.dumps([m for m in messages if m["role"] == "system"]),
            )
            span.set_attribute(
                "gen_ai.input.messages",
                json.dumps([m for m in messages if m["role"] != "system"]),
            )
        # 调用返回后再设置
        span.set_attribute("gen_ai.usage.input_tokens", 120)
        span.set_attribute("gen_ai.usage.output_tokens", 45)
```

这个开关很要紧。规范说，打点实现默认*不应该*采集内容属性，因为消息很大，而且常常含用户 PII。在 dev 和 staging 里直接 `capture_content=True` 跑，不用讲究。在生产里，采样（5% 是个合理的起点），或者把内容放进对象存储、在 span 上留一个引用。最糟的配置是你没得选：内容永远开着（合规风险），或者永远关着（没法调试）。

自动打点能省掉大部分样板代码。Traceloop 的 OpenLLMetry 包了 OpenAI、Anthropic 和 LangChain 的 SDK，自动发出这些属性；它能把数据送到你已经在用的任何 OTLP 后端，包括 Datadog、Honeycomb 和 Grafana Tempo。Arize Phoenix、LangSmith、Langfuse 则在你上面再提供带提示词视角的托管后端。按你现有的栈来选：已经在用 LangChain 就选 LangSmith；想要能自托管的开源方案就选 Langfuse 或 Phoenix；想把 LLM traces 并进现有可观测性栈，就 OpenLLMetry 加你手上的那套。

## 在每个 span 上钉住提示词版本

还有一个属性，你几乎肯定希望它出现在每个生成 span 上，而约定目前还没把它标准化：当时用的是哪个提示词版本。原因是每次事故复盘都会问同一个问题：*回归是出在 v12 还是 v13？* 如果你的 trace 只记了 `gpt-4o` 和一个 token 数，却没记提示词版本的整数，那事后分析就是猜。

Langfuse 和 LangSmith 都用“不可变版本 + 可变标签”的模式来解决。每次编辑提示词就产生一个新的整数版本（不可变）。`production`、`staging`、`experiment-a` 这类标签是指向某个具体版本的可变指针。你的应用代码引用标签，永远不引用整数；上线一次提示词改动，就是把 `production` 从 v12 改指向 v13，不用改代码。回滚就是把标签指回 v12。

关键的巧思在于记什么。trace 记录这次请求实际用到的*整数*（v13），同时记下标签（`production`）。当事故进行中有人改了标签指向，你的 traces 依然说实话，能告诉你每个请求当时命中哪个版本。再配上 `gen_ai.response.model`（它记录供应商实际服务的是哪个模型，与你请求的模型不同），你就能回答“模型为什么看到那个？”的两半：你的提示词版本，和供应商的模型版本，在每一个 span 上都有。

有一个坑值得知道。Langfuse SDK 默认在进程内缓存拉到的提示词 60 秒，采用 stale-while-revalidate 刷新。你改完 `production` 标签之后，已经在跑的应用实例会继续用旧版本，最长可达缓存 TTL，然后再返回一次旧版本，同时后台刷新。所以一次“14:00 部署”的提示词改动，14:02 可能还在服务旧版本。对多数改动这没关系，对安全相关的改动就是问题；这类调用传 `cache_ttl_seconds=0` 绕过缓存，或者用 Langfuse 的 fallback prompt 功能保证可用性。[^5]

## 闭环，端到端

把这些接起来之后，调试一次回归就有了固定形状。用户反馈一个烂答案。你拉出 trace，读出模型收到的那个确切的 `messages` 数组，看到提示词版本（v13）和实际回复的模型（`gpt-4o-2024-11-20`），把这段输入拷进本地复现。你把失败的案例加进黄金样本群，看着样本群得分掉到门禁线以下，它就成了一个真实测试，任何后续修复都必须通过。如果问题在组装逻辑，下一个 PR 的哈希会翻，一条属性测试会指到被破坏的不变量。如果问题在模型行为，样本群门禁会抓住它，并告诉你是把提示词回滚到 v12，还是把模型钉到一个稳定快照。

这个循环就是全部意义。把上下文当成它本来就是的东西——一个有版本、可测试的构建产物——“模型为什么看到那个？”就不再是谜，而是对你 traces 的一次查询。

## 参考文献

[^1]: Tian Pan, "Snapshot Tests Lie When Your Model Is Stochastic," tianpan.co, May 2, 2026. https://tianpan.co/blog/2026-05-02-snapshot-tests-lie-when-model-stochastic

[^2]: Nemanja Ninkovic, "Prompt snapshot testing," ninkovic.dev, May 2025. https://ninkovic.dev/blog/2025/prompt-snapshot-testing

[^3]: Wanqin Ma, Chenyang Yang, and Christian Kastner, "(Why) Is My Prompt Getting Worse? Rethinking Regression Testing for Evolving LLM APIs," CAIN 2024, ACM. https://arxiv.org/abs/2311.11123

[^4]: OpenTelemetry Authors, "Semantic conventions for generative client AI spans," OpenTelemetry Semantic Conventions v1.41.1, June 2026. https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/

[^5]: Langfuse, "Prompt Management: Core Concepts" and "SDK-level prompt caching changelog," langfuse.com, 2024-2025. https://langfuse.com/docs/prompt-management/data-model and https://langfuse.com/changelog/2024-02-05-sdk-level-prompt-caching
