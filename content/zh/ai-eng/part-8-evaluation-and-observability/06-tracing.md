---
title: "链路追踪"
description: "LLM 调用、检索与 agent 循环的 span：gen_ai.* 的 OpenTelemetry 约定，以及厂商私有追踪为何会把你锁死。"
slug: 06-tracing
part: 8
chapter: 6
difficulty: intermediate
prerequisites:
  - part-7-agent-engineering/02-the-agent-loop
  - part-5-context-engineering/05-testing-and-observing-context
tags: [llm, observability]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 链路追踪

用户反馈说，你的客服 agent 在退款窗口这件事上给了一个自信满满的错答案。你打开日志。看到请求 14:03:22 进来，响应 14:03:31 出去，模型是 `gpt-4o`。九秒，一个模型名，和一次关于退款的幻觉。这就是你手上的全部信息。

是检索器拉到了过期文档？改写器在检索之前把用户的问题改坏了？agent 跑了三轮工具调用还是七轮？某次工具调用返回了错误，而模型悄悄把它糊了过去？从一行扁平的日志里，这些问题一个都答不上来。模型只产出了一个可见输出，但它是穿过一棵操作树才走到那儿的，而出问题的正是这棵树。

trace（追踪）让这棵树变得可读。请求里的每个操作变成一个 **span（跨度）**：一条带类型、带时间戳的记录，有名字、时长、父节点和一组属性。span 可以嵌套。根 span 就是用户的请求，它的子节点是 LLM 调用、检索步骤、每次工具执行。在任意后端打开这条 trace，你会看到一张瀑布图：哪一步什么时候跑、花了多久、消耗多少 token、把什么参数传给下一步。幻觉变得可定位了。

## LLM 应用的 span 树

对一个 RAG agent 来说，这棵树的形状是可以预期的。一个根 span 包住整次 agent 调用。下面，子 span 覆盖每一轮循环：一次检索调用、一次 LLM 调用、一次工具执行，等等。如果 agent 循环三次，根下面就有三组子 span。

![一张横向的瀑布式 trace 视图：顶部是一条横跨全宽的靛蓝色长条，标签是“invoke_agent rag-pipeline”；它下面左侧有一条较短的青蓝色条，标签是“retrieval”，紧接着是一条更长的靛蓝色条，标签是“chat gpt-4o”，起点就在检索结束的位置；再往下有一条珊瑚色条，标签是“execute_tool refund_lookup”，标出模型调用工具的那一刻；几个小的属性标注分别指向“input_tokens: 312”、“data_source.id: docs-v3”、“tool.name: refund_lookup”](../../../ai-eng/part-8-evaluation-and-observability/images/06-rag-trace-tree.png)
*一条 trace 就是一张带类型的瀑布图：父 agent span 包住它触发的每次检索、LLM 调用和工具执行，属性则记录每一步到底做了什么。*

从操作到 span 的映射，正是 OpenTelemetry GenAI 语义约定（semantic conventions）标准化的内容。[^1] AI 应用里每一类操作都有各自的 span 名和一组属性：

- **LLM 调用**变成 `chat {model}` span，带 `gen_ai.operation.name = "chat"`、`gen_ai.provider.name`、`gen_ai.request.model` 和各项用量计数。span 类型是 `CLIENT`（模型跑在另一个进程里）。
- **检索步骤**变成 `retrieval {data_source}` span，带 `gen_ai.operation.name = "retrieval"` 和 `gen_ai.data_source.id`。检索到的文档和查询文本属于需主动开启的属性。
- **工具执行**变成 `execute_tool {tool_name}` span，带 `gen_ai.tool.name`、`gen_ai.tool.type` 和 `gen_ai.tool.call.id`。span 类型是 `INTERNAL`。
- **agent 调用**变成 `invoke_agent {agent_name}` span，包住其他一切。进程内 agent 的类型是 `INTERNAL`，调用远程 agent 服务时是 `CLIENT`。

名字很重要。后端靠它们渲染瀑布图、按操作类型分组、计算聚合指标。如果你把 span 叫成 `"step 3"`，等于把这些机制全扔了。

在代码里，每个 span 是一个 Python 上下文管理器。嵌套是 `with` 块白送的：在另一个 span 里启动的任何 span，都会通过 OTel 的上下文传播成为它的子节点。

```python
from opentelemetry import trace

tracer = trace.get_tracer("rag-agent")

def rag_pipeline(query: str) -> str:
    with tracer.start_as_current_span("invoke_agent rag-pipeline") as agent:
        agent.set_attribute("gen_ai.operation.name", "invoke_agent")
        agent.set_attribute("gen_ai.agent.name", "rag-pipeline")

        with tracer.start_as_current_span("retrieval docs-v3") as ret:
            ret.set_attribute("gen_ai.operation.name", "retrieval")
            ret.set_attribute("gen_ai.data_source.id", "docs-v3")
            docs = ["..."]  # 实际的向量检索

        with tracer.start_as_current_span("chat gpt-4o") as llm:
            llm.set_attribute("gen_ai.operation.name", "chat")
            llm.set_attribute("gen_ai.provider.name", "openai")
            llm.set_attribute("gen_ai.request.model", "gpt-4o")
            answer = "..."  # 实际的 LLM 调用
            llm.set_attribute("gen_ai.usage.input_tokens", 312)
            llm.set_attribute("gen_ai.usage.output_tokens", 97)

        return answer
```

这就是整套埋点（instrumentation）模式。实际用的时候你会把它包进一个小 helper，或者直接用自动埋点（OpenLLMetry、OpenInference）——它们在 import 时 monkey-patch 你的 provider SDK，不用碰调用点就能发出这些 span。

## agent 循环的嵌套在哪里咬到你

agent 循环是这个模型变得有意思的地方。agent 跑一个 think-act-observe 循环，直到它自认干完了；每次迭代产出一个 LLM 调用和零个或多个工具调用。最自然的埋点做法是让 `invoke_agent` span 整轮开着，让每次迭代的 span 嵌在它下面。

对一个几秒钟就返回的 agent，这没问题。对长时间运行的 agent，父 span 要到整轮结束才关闭。两个地方会坏。OTLP exporter 有批处理超时，开得比超时还久的 span 会被丢弃或截断，你最后拿到一堆孤儿子 span 而没有根。[^2] 而且 agent 停下来之前，你在后端看不到任何部分进度，因为多数后端只在 trace 完整之后才渲染。

解法是压平一层。把父 agent span 发成一个短包装，只记录这轮运行的身份（`gen_ai.agent.name`、`gen_ai.agent.id`、用户可见的任务），然后给每一轮迭代开一个新的子 span。每次迭代在它的 LLM 调用和工具调用结束后关闭、导出，实时出现在后端。整次 agent 运行仍然可以作为一棵树查询，因为每一轮迭代共享同一个 `traceId`；你只是不再假装它是一个操作。

任何分布式追踪教程里都有的那个异步上下文警告，在这里同样适用，而且比别处更疼：如果你把工具调用丢进线程池或 `asyncio.create_task` 而不传播上下文，子 span 会开出一条全新的 trace。两个后端、两个视图、没有父节点。线程池用 `contextvars.copy_context()`，发后台任务时把当前上下文显式传进去。

## 为什么厂商私有的追踪是个陷阱

每个 LLM 平台都在发自己的追踪 SDK。LangChain 有回调处理器，一路发到 LangSmith。Anthropic 有 tracing hooks。OpenAI 的 Traces API 有自己那一套。每一个都是两行配置就自动接上，认得你框架的调用，在它们的 UI 里给出可用的视图。做原型的时候摩擦几乎为零，UI 还往往比你自己搭的好看。

原型阶段就用它们。然后，在真实用户的数据流过去之前，换到 OTel。

理由是可移植性（portability），而弄错的代价，只有在你想要换的那天才感觉得到。厂商 SDK 用一种私有线格式、一套私有属性名，导出到一个后端。换后端意味着把代码库里每个调用点重新埋一遍，而你等得越久，依赖回调的代码就越多，要重写的也越多。更糟的是，你的 AI 应用不是单独跑的：它身边有 web 服务器、数据库、队列、后台 worker，这些大概率已经在 OTel 上了。用私有 AI SDK，喂给你检索调用的那条数据库查询活在一个 trace 系统里，消费它输出的 LLM 调用活在另一个里。你没法把整个请求看成单独一棵树，而那就是追踪的全部意义。

OTel 用一个线格式（OTLP）和一个属性 schema（`gen_ai.*`）解决这个问题，现在每个正经后端都吃这一套。Langfuse 在 `/api/public/otel/v1/traces` 暴露 OTLP 端点，把这些约定映射到自己的数据模型上。[^3] Arize Phoenix 通过 `phoenix.otel` 包装层就是 OTel 原生的。LangSmith 在 2024 年 12 月加了 OTLP 端点，2025 年 3 月端到端 OTel 支持正式可用，此前多年只支持回调。[^4] Datadog、Honeycomb、Grafana Tempo 以及任何原生 OTel 后端收的是同一份 OTLP 流量。你埋点一次，路由到你想要的后端，改个配置就换后端，而数据库 span 白送地和 LLM span 共享一棵树。

```python
import base64, os
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# 同样的埋点，不同的后端：改端点，不改 span。
auth = base64.b64encode(
    f"{os.environ['LANGFUSE_PUBLIC_KEY']}:{os.environ['LANGFUSE_SECRET_KEY']}".encode()
).decode()

exporter = OTLPSpanExporter(
    endpoint="https://cloud.langfuse.com/api/public/otel/v1/traces",
    headers={"Authorization": f"Basic {auth}"},
)
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
```

值得认真对待的异议是：`gen_ai.*` 命名空间到 2026 年年中是语义约定 1.41.1 版，里面每个属性的稳定性都标着 `Development`，不是 `Stable`。[^1] 名字还可能会变。semconv v1.36.0 之前的既有埋点发的是旧的 `gen_ai.system` 属性，而不是现在的 `gen_ai.provider.name`，规范明确要求那些埋点不要自动升级。设 `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` 来选择最新名字，或者在 OTel Collector 用属性处理器做归一化。这就是抢跑的代价，比跳过标准带来的锁定成本要小。

## span 上放什么，不放什么

这些约定把属性分成三档要求：必需、推荐、需主动开启。必需属性出现在每个 span 上（`gen_ai.operation.name`、`gen_ai.provider.name`）。推荐属性出现在每个值可得的 span 上（`gen_ai.usage.input_tokens`、`gen_ai.usage.output_tokens`、`gen_ai.response.finish_reasons`、`gen_ai.request.temperature`）。需主动开启的那些，是你要刻意决定是否放的，因为它们携带用户内容：`gen_ai.input.messages`、`gen_ai.output.messages`、`gen_ai.system_instructions`、`gen_ai.tool.call.arguments`、`gen_ai.tool.call.result`、`gen_ai.retrieval.query.text`、`gen_ai.retrieval.documents`。

把需主动开启的那批当成隐私决策，而不是调试便利。把用户的 prompt 发给第三方追踪后端，和发给第三方 API 是同一条数据流，在 GDPR、HIPAA 或你所在辖区的数据驻留规则下，合规含义完全一样。生产环境的默认值是关。如果为了查问题确实需要内容，那就采样（1%~5% 是个站得住脚的起点），在 OTel Collector 用处理器脱敏，或者把内容存进自己的对象存储、只在 span 上挂一个引用。[测试与观测上下文](../part-5-context-engineering/05-testing-and-observing-context.md) 一章讲了开发模式的做法：staging 开着采集，生产走闸门。

推荐属性是不带隐私风险就有回报的那批。每个 span 上的 token 数让你按请求、按用户、按路由归因成本。finish reason 让你发现 `length`（输出被截断）或 `content_filter` 突然飙升。LLM span 与检索 span 的延迟对比，告诉你 p99 问题出在模型还是向量库。这些就是你凌晨三点真的会跑的查询——那个给错退款答案的 agent 变成一张真实工单时，你第一个打开的就是 trace。

## 参考文献

[^1]: OpenTelemetry，“Semantic conventions for generative client AI spans”（semconv 1.41.1，状态：Development），opentelemetry.io，2026 年 6 月访问。https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/

[^2]: OpenTelemetry，“Semantic Conventions for GenAI agent and framework spans”（semconv 1.41.1），opentelemetry.io，2026 年 6 月访问。https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/

[^3]: Langfuse，“OpenTelemetry (OTEL) for LLM Observability”，langfuse.com/docs，2026 年 6 月访问。https://langfuse.com/docs/opentelemetry/get-started

[^4]: LangChain Team，“Introducing OpenTelemetry support for LangSmith”，blog.langchain.com，2024 年 12 月 9 日。https://blog.langchain.com/opentelemetry-langsmith/
