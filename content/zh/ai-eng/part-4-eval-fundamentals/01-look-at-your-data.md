---
title: "先看你的数据"
description: "LLM 评测里投入产出比最高的一个习惯：在自动化任何东西之前，先手工读 trace，建出一套失败分类体系。"
slug: 01-look-at-your-data
part: 4
chapter: 1
difficulty: beginner
prerequisites:
  - 00-why-you-cant-ship-without-evals
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-08-31
---

# 先看你的数据

一位顾问走进团队办公室，只问了一个问题：“你们怎么衡量这个东西能不能用，能演示一下吗？”团队打开一块仪表盘。连贯性：4.1。流畅度：4.3。有用性：3.9。大多数周这些数字都在涨。客服工单队列却一直在堆。

这就是本章要讲的失效。这个团队不懒。他们第一天就装了评测平台，也把各项指标接上了。他们漏掉的是任何指标产生意义之前那段不好看的活：坐下来，把一百条真实 trace 从头读到尾。

截至 2025 年 3 月，在 30 多个生产环境的 AI 项目里，Hamel Husain 几乎每次都看到同一种模式。[^1] 团队会去抓通用指标（“幻觉”“毒性”“连贯性”），因为平台自带的就这些。通用指标和任何一个具体产品的具体失效都不相关。你的准确率从 3.72 涨到 4.20，而你其实并不知道系统有没有变好。[^2]

解法比 LLM 还老。就是定性研究的那个循环：读数据、写笔记、把笔记归成类别、数每个类别的量、先修最常见的那个。在真实产品上做扎实，这是 AI 工程师能做的杠杆最高的一件事。NurtureBoss 是一款公寓租赁助手，正是靠这套流程发现相对日期解析（“安排两周后看房”）有三分之二的时候是失败的。做了针对性修复之后，这一个类别从 33% 成功率升到 95%。[^1] 没有任何现成指标能把它暴露出来。

## 一条 trace 就是工作单元

一条 trace 是一次交互的完整记录：系统提示词、用户输入、任何检索到的上下文、任何工具调用，以及模型的输出。多轮对话里，一条 trace 就是整段对话。领域专家判断“这次做得好不好”需要读的全部内容，就是那个单元。

别把它和分布式系统里的 trace 搞混。那种 trace 是为了找崩溃和超时。LLM 的失效不会崩溃。模型返回 200 OK 和一段流畅的文字，内容却在悄悄出错、跑偏语气，或者客气地拒绝用户真正想要的东西。信号在内容里，不在状态码里。[^3]

起步不需要可观测性基础设施。合适的起点是一张表格或者一个平铺文件，一条 trace 一行，几列：

```python
import datetime, uuid
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Trace:
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: str = field(default_factory=lambda: datetime.datetime.utcnow().isoformat())
    session_id: Optional[str] = None
    system_prompt: str = ""
    user_input: str = ""
    assistant_output: str = ""
    model: str = ""
    # 手工评审时填写：
    label: Optional[str] = None         # "pass" | "fail"
    note: Optional[str] = None          # 自由文本，哪里出了问题
    category: Optional[str] = None      # 轴心编码之后填写
```

前六个字段能干净地对应到 OpenTelemetry 的 GenAI 语义约定（`gen_ai.request.model`、`gen_ai.conversation.id` 等，截至 2026 年 6 月为 semconv 1.41.1），所以以后接入 tracing 库时，这套 schema 不用重写就能升级。[^4] 后三个字段才是今天真正重要的。在人来填之前，它们是空的。

这一段没人能自动化绕过去。人读 trace，人写笔记。

## 开放式编码：先读，再写日记

开放式编码（open coding）是过数据的第一遍。一次打开一条 trace，从头读到尾，把任何不对的地方用自由文本记下来。先不要分类体系，不要类别，只有笔记。Husain 把这一步叫“写日记”（journaling），词是从扎根理论的定性研究里原样借来的。[^5]

有几条规则能帮你避开最常见的错误。

- **读的人应该是领域专家，不是工程师。** 一条客服 trace 里，用户在已知故障之后抱怨“账单”，这不是模型失效，那是对一个异常用户状态的正确回答。工程师冷读会把它判错。最好的标注者是懂产品又懂用户的那个人。[^3] 工程师在这里的职责是工具，不是判断。
- **只记*第一个*失效，不要记全部。** 在多轮或 agentic trace 里，早期的错误会级联。第 2 轮一个错误的工具调用，导致第 3 轮拿到错误的上下文，又导致第 4 轮给出错误的最终答案。如果三个都记，你的分类体系会把同一个根因重复计两三次，“最终答案错误”这个桶会爆掉，而上游的问题被藏起来。[^5]
- **用自己的话写。** 不要试图先分类。“忘了用户三轮前说过自己吃素”比“记忆失效”是一条更好的笔记。类别要从笔记里长出来，不是笔记从类别里倒推。
- **二元标签，外加笔记。** 通过或失败。不要 1 到 5 分的量表。Husain 和 Shankar 都认为二元标签逼着评审做出真正的判断，细微之处留在自由文本笔记里，而不是塞进一个假精确的数字。[^1][^6]

这个阶段用一个标注者，不要用委员会。在给数千名工程师和 PM 教过评测之后，Husain 的默认做法是他所谓的“仁慈独裁者”：一个领域专家，他的判断定义了标准。[^7] 标注者间一致性是后面的事，那时你要让自动评审对齐人类标签。在开放式编码阶段，两个没有共同评分标准的评审者，只会围绕一套还不存在的标准产出噪声。

## 轴心编码：把笔记变成分类体系

手里有一叠笔记之后，第二遍把相似的笔记归到有名字的类别里。这就是轴心编码（axial coding）。产出是一套小分类体系，通常五到十个标签，再加上每个类别落了多少条 trace 的计数。那个计数是之后每一个决策的依据。

![漏斗图：原始生产 trace 流入一叠自由文本标注笔记，笔记归组成三个带计数的具名桶，一支箭头指向右侧的“写测试”目标](../../../ai-eng/part-4-eval-fundamentals/images/01-error-analysis-funnel.png)
*开放式编码产出笔记；轴心编码把它们归组；计数告诉你先修哪个失效。*

分类体系要从你真写下的笔记自下而上长出来。不要从一个通用清单（“幻觉、毒性、连贯性”）开始，再把数据硬塞进去；团队就是这样最后测的东西跟自己的产品无关。[^1] 归组这一步可以让 LLM 帮忙，因为输入是一叠短文本笔记，聚起来很干净。便宜的做法就是一句提示词：“这里有 80 条开放式编码笔记；提出 5 到 10 个有名字的类别，并把每条笔记分到其中一个。”提出的分类体系仍然要人过一遍才算接受，而这个顺序才是关键：人审的是归类结果，不是原始标签。[^5]

然后数。电子表格里一张透视表就够了。NurtureBoss 就是这么做的；三个类别占了全部失效的 60% 以上（对话流程、转人工、日期处理），其中日期处理是最大的一块。[^1] 计数把“这个机器人不行”这种模糊感觉，变成一个具体的、可修的数字。它也告诉你什么*不该*修：一百条里只有三条的类别，不是这周该投入的地方。

这些类别是驱动下一章的东西。每一个都会变成一个自动检查的目标。“日期处理”变成一组相对日期输入、配上已知正确输出的测试集。“转人工失败”变成一条断言：该调用的工具被调用了。分类体系是读散文和写代码之间的桥。

## 读到什么时候停

读到第三条 trace 之后，自然会冒出的问题是“这些我得读多少条？”诱惑是早点停。二十条 trace 感觉已经很多了，尤其每条要花五分钟。

规则借自扎根理论里的*理论饱和*（theoretical saturation）：一直读到连续 20 条 trace 都没有带来新类别，同时总量不低于 100 条。[^5] 不到 100 条，你只看到了明显的失效，漏掉了长尾。停在 30 条，就是这样让一个你从没见过的失效模式带着零测试覆盖上线，六周后以一条客户投诉的形式冒出来，而你没有任何评测能接住它。

这条规则可以直接写成代码：

```python
from typing import List, Set

def reached_saturation(
    category_history: List[Set[str]],
    window: int = 20,
    min_reviewed: int = 100,
) -> bool:
    """Stop when the last `window` traces introduced no new categories,
    after at least `min_reviewed` total traces."""
    total = len(category_history)
    if total < min_reviewed or total < window:
        return False
    earlier = set().union(*category_history[:-window])
    recent = set().union(*category_history[-window:])
    return len(recent - earlier) == 0
```

这不是一次性工作。每次重要的提示词改动、模型升级或新的用户群体之后，失效空间都会移动，分类体系要重新过一遍。生产环境里合理的节奏是每周从线上流量里读 20 到 30 条新 trace，盯着那些套不进任何现有桶的类别。Shankar 把这套叫“数据飞轮”：评测、监控和持续改进互相喂。[^6] 你今天定下的指标集，不会是你六个月后的指标集，而这正是重点。

## 标准漂移是特性，不是缺陷

读到第 40 条 trace 附近，会有件不太舒服的事发生。你意识到前十条你现在会判得不一样。整个下午都在标的那位评审，已经没法完全解释为什么第一场的标注和第三场的不同。本能反应是把它叫作不一致，担心数据被污染了。

并不是。Shankar 和同事在 2024 年的一篇 CHI 论文里把这个现象命名为*标准漂移*（criteria drift）：有些评测标准“依赖于实际观察到的具体 LLM 输出”，事先无法知道。[^8] 打分这件事本身教会你该按什么打分。读一百条输出的过程，会暴露出你的规范从没预料到的边界情况，而随着语料自己展开，你的标准也在变锋利。这不是方法论上的缺陷，这是方法论在起作用。

有一个真实的生产观察。Husain 和 Honeycomb 的 Phillip Carter 一起做 Query Assistant 功能时，Carter 在过程中注意到：“看到 LLM 怎么拆解它的推理，让我意识到我在判断某些边界情况时一直不一致。”[^1] 他不是马虎。他是在通过使用这套标准，学会自己的标准该是什么。

实际应对很简单。把评分标准当成一份活文档。像给代码做版本一样给它做版本。标准变锋利的时候，在早期 trace 的随机样本上回填标签，让新旧标注保持可比。并且预期这套标准在头几周会一直动；如果第三天它就稳定了，你大概还没读够数据。

## 先用电子表格，难受了再上查看器

到这里，诱惑是去挑一个平台。忍住一周。这里最有影响力的单件基础设施不是某个厂商，而是那个让领域专家两秒内标完一条 trace、不用切标签页的界面。Husain 在几十个项目里造过这类工具之后的说法是：有趁手数据查看器的团队，迭代速度大约是没有的团队的 10 倍，而这个查看器用 AI 辅助开发几个小时就能做出来。[^1]

一个能用的起点真的就是一张电子表格。列：`trace_id`、`input`、`output`、`note`、`category`。Husain 在 2024 年 12 月的 office hours 里看过的那个照护短信应用，就是这么起步的，后面用 Azure OpenAI 和 Helicone 做日志。[^2] 电子表格在 trace 涉及单元格装不下的东西时会失效：很长的检索文档、工具调用的入参与返回、评审需要和对话并排看的 CRM 上下文。那时候你就该自己做点东西。

自建查看器不需要多厉害。它需要四件事：

- **一屏看全。** trace、检索到的上下文、工具调用，以及评审判断正确性需要的任何业务数据。切标签页是扼杀标注速度的元凶。
- **一键二元标签。** 通过或失败，不是一个七项下拉菜单。
- **自由文本笔记框。** 永远在，永远保存。
- **键盘导航。** 下一条、上一条、打标签、保存，全程不碰鼠标。

托管工具（LangSmith、Braintrust、Arize Phoenix、Langfuse）都提供 trace 查看器和标注流程。它们的差别主要在渲染领域特定上下文有多干净，以及非工程师能不能不经培训就用。[^9] 在你过了电子表格阶段、又还没开始往里砸几周自建之前，挑一个。自建还是买，这个决定确实很接近；你唯一不能搞错的决定是*至少得有个查看器*。

这个循环走完，你手上有一份很小的、排好序的具名失效类别清单加计数，还有一个大约 100 条 trace 的数据集，每行带一个二元标签、一条笔记、一个类别。这个数据集就是[你的第一套评测集](./02-your-first-eval-set.md)的起点：失效类别告诉你该测什么，标注好的 trace 给你第一批用来测的例子。花一个下午读散文这件不讨好的活，正是让你之后写下的每一个指标都有意义的东西。

## 参考文献

[^1]: Hamel Husain, "A Field Guide to Rapidly Improving AI Products," hamel.dev, March 24, 2025. https://hamel.dev/blog/posts/field-guide/
[^2]: Hamel Husain, "Evals: Doing Error Analysis Before Writing Tests," hamel.dev (Open Office Hours notes), December 21, 2024. https://hamel.dev/notes/llm/officehours/erroranalysis.html
[^3]: Hamel Husain, "Your AI Product Needs Evals," hamel.dev, March 29, 2024. https://hamel.dev/blog/posts/evals/
[^4]: OpenTelemetry Authors, "Semantic conventions for generative client AI spans," Semantic Conventions v1.41.1, opentelemetry.io, June 2026. https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans
[^5]: Hamel Husain, "Q: Why is 'error analysis' so important in LLM evals, and how is it performed?" AI Evals FAQ, hamel.dev, May 5, 2026. https://hamel.dev/blog/posts/evals-faq/why-is-error-analysis-so-important-in-llm-evals-and-how-is-it-performed.html
[^6]: Shreya Shankar, "Data Flywheels for LLM Applications," sh-reya.com, July 1, 2024. https://sh-reya.com/blog/ai-engineering-flywheel/
[^7]: Hamel Husain and Shreya Shankar, "LLM Evals: Everything You Need to Know," hamel.dev, May 2026. https://hamel.dev/blog/posts/evals-faq/
[^8]: Shreya Shankar, J.D. Zamfirescu-Pereira, Bjorn Hartmann, Aditya G. Parameswaran, Ian Arawjo, "Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences," arXiv:2404.12272, April 18, 2024 (ACM UIST 2024). https://arxiv.org/abs/2404.12272
[^9]: Hamel Husain, "Selecting The Right AI Evals Tool," hamel.dev, July 2025. https://hamel.dev/blog/posts/eval-tools
