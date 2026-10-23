---
title: "采样与 temperature"
description: "temperature 和 top_p 如何重塑模型的概率分布、如何按任务挑选取值，以及为什么 temperature 0 并不等于确定性保证。"
slug: 02-sampling-and-temperature
part: 1
chapter: 2
difficulty: beginner
prerequisites:
  - part-0-start-here/02-thinking-in-probabilistic-systems
tags: [llm]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 采样与 temperature

模型并不是在“挑”下一个词。它给词表里的每个词打一个分，把这些分数变成概率，然后抽一个出来。temperature 和 top_p 就是抽样之前重塑这个分布的两个旋钮。你要是把它们当成“创造力拨盘”，最后交付的会是一个爱胡编的分类器，和一个说话像自动售货机的聊天机器人。

[在概率系统里思考](../part-0-start-here/02-thinking-in-probabilistic-systems.md) 已经论证过：调用一次 LLM 是从分布里抽样，不是调用一个函数。这一章讲的就是怎么有意识地塑造那个分布，一个任务一个任务地调。

## temperature 重塑这条曲线

先从原始分数说起，它叫 logits。这里有三个 token，logits 是 `[3.0, 1.0, 0.5]`。temperature 在做 softmax 之前，把每个 logit 都除以同一个数：

```python
import math

def softmax_with_temperature(logits: list[float], temperature: float) -> list[float]:
    if temperature <= 0:
        raise ValueError("temperature must be > 0")
    scaled = [l / temperature for l in logits]
    max_l = max(scaled)
    exps = [math.exp(x - max_l) for x in scaled]
    total = sum(exps)
    return [e / total for e in exps]

logits = [3.0, 1.0, 0.5]
for temp in [0.2, 1.0, 2.0]:
    print(f"T={temp}: {[round(p, 3) for p in softmax_with_temperature(logits, temp)]}")
# T=0.2: [1.0, 0.0, 0.0]        <- 一个 token 拿走全部概率
# T=1.0: [0.821, 0.111, 0.067]  <- 模型本来的分布形状
# T=2.0: [0.604, 0.222, 0.173]  <- 被摊平；冷门 token 也能分到票
```

除以一个小的数，logits 之间的差距被拉大，最高分那个 token 就吞掉了几乎全部概率。除以一个大的数，差距被压缩，弱势 token 又爬回竞争圈。机制就是这么简单：temperature 小于 1 会锐化尖峰，大于 1 会摊平。

![三张并排柱状图，展示同一组 token 概率在三个 temperature 下的形态：T=0.2 时是一根很高的尖峰，T=1.0 时是一根占主导的柱子加一条尾巴，T=2.0 时各柱高度大致相当](../../../ai-eng/part-1-how-models-behave/images/02-temperature-shapes-the-curve.png)
*同一组 logits，三个 temperature。低温把赌注集中在一个 token 上；高温把它分散到许多 token 上。*

temperature 0 是这套机制的极限：temperature 趋近 0 时，最高分 token 的概率趋近 1。多数 API 直接把它退化成 `argmax`，每一步都选分数最高的那个 token。当你要的是“一个答案”而不是“一批答案”时，用的就是这个设置。

各家允许的范围不一样，而这些差异在生产环境里真的会咬人：

| 提供方 | temperature 范围 | 默认值 | 统计时间 |
|---|---|---|---|
| OpenAI（Chat Completions、Responses） | 0 到 2 | 1 | 2026 年 6 月[^1] |
| Anthropic Claude Haiku 4.5、Sonnet 4.5、Sonnet 4.6 | 0 到 1 | 1 | 2026 年 6 月[^2] |
| Anthropic Claude Opus 4.7+（4.8、Fable 5、Mythos 5） | 参数被拒绝（HTTP 400） | 不适用 | 2026 年 6 月[^2] |
| Google Gemini（GenerationConfig） | 0 到 2 | 随模型不同 | 2026 年 6 月[^3] |

Anthropic 最后一行不是笔误，它直接影响你写多模型代码的方式，下面细说。

## top_p 剪掉长尾

temperature 会重塑每一根柱子。top_p 又叫核心采样（nucleus sampling），它做了第二刀：把不靠谱的长尾 token 整个扔掉，然后对剩下的重新归一化。

规则是“保留概率加起来刚好到 `p` 的最小 token 集合，其余丢掉”：

```python
def top_p_filter(token_probs, top_p):
    if not (0.0 < top_p <= 1.0):
        raise ValueError("top_p must be in (0, 1]")
    cumulative, nucleus = 0.0, []
    for token, prob in sorted(token_probs, key=lambda x: -x[1]):
        nucleus.append((token, prob))
        cumulative += prob
        if cumulative >= top_p:
            break
    total = sum(p for _, p in nucleus)
    return [(t, p / total) for t, p in nucleus]

tokens = [("cat", 0.40), ("dog", 0.30), ("bird", 0.15), ("fish", 0.10), ("ant", 0.05)]
print(top_p_filter(tokens, top_p=0.8))
# [('cat', 0.471), ('dog', 0.353), ('bird', 0.176)]
# cat + dog + bird 累计到 0.85，越过了 0.80，所以在这里停下。
# fish 和 ant 在抽样发生之前就被切掉了。
```

妙处在于这个切割是自适应的。当模型确信下一个词是 “the” 时，即使 `top_p=0.95`，核心集合里也可能只有一个 token。当五十种续写都说得通时，核心集合会变宽，把它们都装进去。top_p 跟踪的是模型每一步的真实把握程度，而不是永远从一个固定大小的池子里抽样。

在 OpenAI、Anthropic 的 Sonnet 和 Haiku、以及 Gemini 上，top_p 的取值区间都是 0 到 1，默认值 1 表示“什么都不切”。[^1][^2][^3] Gemini 另外提供 `topK`，一个整数，限死最多能有多少 token 进入候选池；OpenAI 则完全不暴露 top-k。[^3]

> [!IMPORTANT]
> **只调 temperature 或 top_p 其中一个，别两个一起调。** OpenAI 和 Anthropic 的文档都这么写，原因也很机械：temperature 先重塑曲线，top_p 再从这个已经被重塑过的曲线上切尾巴。你在 temperature 0.3 下切掉的尾巴，和在 temperature 1.0 下切掉的完全是两回事，两个旋钮的相互作用没有人能靠眼睛推算。[^1][^2] 选一个维度调，另一个留在默认值。

## 按任务选设置，别靠感觉

这些是方差控制，不是质量控制的旋钮。低 temperature 不会把错答案变成对答案，它只是让模型对自己已经相信的东西更自信。所以你要根据“输出有多样化的代价”来选，而不是根据任务“感觉多有创意”。

凡是能判分的任务，都从 temperature 0 起步。只有当输出多样性带来真实的产品价值时，才往上调。下表的默认值来自各家官方建议与实践者共识：[^4][^5]

| 任务 | temperature | 原因 |
|---|---|---|
| 抽取、分类、JSON 输出 | 0 | 你要的是唯一一个最有把握的答案；再配上结构化输出 schema 把剩余的方差也消灭掉。 |
| 代码生成 | 0 到 0.2 | 少一点意外的代码通常就是正确的代码。OpenAI 自己也建议确定性任务用低 temperature。[^4] |
| 基于检索的问答（RAG） | 0 到 0.3 | 让模型贴着检索到的事实；调高了就容易一本正经地编。 |
| 对话聊天、客服 | 0.7 到 1.0 | 多样性足以避开机械感，又不至于跑题。 |
| 创意写作、头脑风暴 | 1.0 到 1.5 | 多样性本身就是目的；连贯性可以松一点。 |

这张表下面有两条注意事项。

第一，是那条异议。有些工程师认为 temperature 0 会让聊天带上一种平板的“机器人腔”，即使用户的使用感受变差，哪怕准确率是提升的。截至 2026 年 6 月，双方都没有强的公开 A/B 数据支撑。稳妥的做法是从低开始、做度量，只有当真实指标（而不是直觉）说明这种平板感确实带来损失时，才把 temperature 调上去。[^5]

第二，唯一值得背下来的 top_p 规则。如果你真要把 temperature 推到 1.0 以上，就配一个 0.9 到 0.95 左右的 `top_p`，把不可靠的长尾切掉。少了这一刀，高 temperature 会让概率接近零的 token 也有机会被选中；而一旦某个奇怪的 token 落地，它会把后面几个 token 都往更奇怪的方向带，退化式的重复循环就是这么来的。这条要拿捏着用：它只基于一个一手来源，即 Holtzman 等人 2020 年的核心采样论文，外加实践者的印证。它是可用的经验法则，不是定律。[^6]

## temperature 0 不是确定性开关

这里有个陷阱，会让你的评测变得时灵时不灵。把 temperature 设成 0 只是去掉了*采样*带来的随机性，并不能让同一份输出在多次调用之间复现。同一个 prompt、temperature 0，两次运行可能返回不同的文本，既没有 bug，也没有改代码。

三种机制在推动这种漂移，截至 2026 年中都仍然存在：

- **浮点加法不满足结合律。** GPU 并行地对部分结果求和，而这个求和的顺序会随 batch size、GPU 数量和硬件代际变化。`(a + b) + c` 并不逐位等于 `a + (b + c)`，于是某个 logit 会挪动一丝。当两个最高分 token 几乎打平时，这一丝就决定了谁赢，之后每个 token 都跟着分叉。[^7]
- **动态批处理。** 云厂商会把你的请求和同一瞬间到达的其他请求打进一个 batch。batch 组成不同，求和顺序就不同，舍入结果就不同，落到哪个 token 上也就可能不同。[^8]
- **混合专家路由（MoE）。** 把 token 路由到专家子网络的模型，在负载均衡的压力下可能给出不同的路由，从而改变同一上下文里同一个 token 的 logits。[^8]

这些数字并不微妙。Yuan 等人（2025）在 bfloat16 贪婪解码下测量了一个推理模型（DeepSeek-R1-Distill-Qwen-7B），发现仅靠改变 GPU 数量和 batch size、采样完全关闭，**基准准确率就波动高达 9%**，**响应长度差异最多可达 9,000 个 token**。[^7]

OpenAI 对此很坦诚。它的 API 提供 `seed` 参数，也会返回 `system_fingerprint`，但文档只承诺“尽力而为”（best effort）的确定性，而且只要后端配置在你脚下变了，指纹就会变。即便固定了同样的 seed 和 temperature 0，指纹一挪，你照样可能拿到不同的输出。[^9]

这正是为什么你的评测必须判断语义，而不是比对字符串。一个把输出和标准字符串做 diff 的评测，会在本地通过、在 CI 失败、重跑又通过，什么都没改。[断言与单元测试](../../../ai-eng/part-4-eval-fundamentals/03-assertions-and-unit-tests.md) 那一章会教你写能容忍这种表层差异的评测；现在就要带走的规则是：断言一个性质（“能解析成带 `city` 键的 JSON”），而不是断言一模一样的字符串。

## 参数版图正在变动

这个话题里最新的意外：Anthropic 最强的几个模型压根不再接受这些参数了。截至 2026 年 6 月，向 Claude Opus 4.7、Opus 4.8、Fable 5 或 Mythos 5 发送 `temperature`、`top_p` 或 `top_k`，会返回 HTTP 400 错误。SDK 的类型定义为了向后兼容仍然列着这些字段，于是你的代码类型检查一切正常，到了生产环境在服务端炸掉。[^2]

Anthropic 的迁移指南把修复方式和背后的理念说得很直白：“最安全的迁移方式是彻底从请求体里省掉这些参数。在 Claude Opus 4.7 上，引导模型行为推荐用 prompt。”[^2] 理由是：最新的模型被训练成从 prompt 里接受行为指导，而不是靠推理阶段对分布做塑造。Part 3 讲 prompt 工程的那几章，讲的就是怎么用文字——而不是一个 temperature 旋钮——来控制语气、正式程度和多样性。

对多模型代码来说，安全的做法是按模型查表，而不是把一个参数塞给“任何模型”：

```python
def get_sampling_params(task_type: str, provider: str) -> dict:
    # provider "anthropic_claude4x" 指 Opus 4.7+，它会拒绝采样参数。
    defaults = {
        "deterministic": {
            "openai":             {"temperature": 0.0, "seed": 42},
            "anthropic_claude4x": {},                  # 留空；用 prompt 引导
            "gemini":             {"temperature": 0.0, "seed": 42},
        },
        "conversational": {
            "openai":             {"temperature": 0.7},
            "anthropic_claude4x": {},
            "gemini":             {"temperature": 0.7, "topP": 0.9},
        },
        "creative": {
            "openai":             {"temperature": 1.0},
            "anthropic_claude4x": {},
            "gemini":             {"temperature": 1.4, "topP": 0.95},
        },
    }
    return defaults.get(task_type, {}).get(provider, {})
```

> [!WARNING]
> **一个把 `temperature` 透传给“别名指向的那个模型”的网关，就是一颗随时会响的 400。** 模型别名会漂到更新的版本，等到你那个别名指向 Opus 4.7+ 的那天，所有设置采样参数的请求都开始在线上失败，而你的集成测试打的是老模型，依旧全绿。构建请求体之前，先拿模型 ID 对一份已知“支持采样参数”的名单核一遍，不支持就把参数剥掉。[^2]

## 设一次，然后别再动

把决策按顺序走一遍，这些旋钮你就能有意识地设，而不是抄别人那个神奇数字：

1. **给任务命名。** 可判分的（抽取、代码、RAG），还是开放式的（聊天、创意）？
2. **按任务而不是按心情定 temperature。** 可判分的 0，聊天 0.7 到 1.0，创意 1.0 到 1.5。只有当你超过 1.0 时才去碰 top_p，碰了就别再动 temperature。
3. **先看模型档位。** 在 Claude Opus 4.7+ 上你什么都不发，用 prompt 引导。其他地方，设你选的那一个旋钮。
4. **永远别假设 temperature 0 会复现。** 评测要设计成判断语义。如果你确实需要字节级可复现来做回归测试，就锁死 seed，并且在 `system_fingerprint` 没变时才允许比对。

这些设置后面也会接进本部分末尾的模型 playground，所以你在那里暴露的控件就是 temperature、top_p，以及一个知道哪些模型接受哪些参数的模型选择器。

## 参考文献

[^1]: OpenAI, "Create chat completion, API Reference"（字段：temperature、top_p、frequency_penalty、presence_penalty）, OpenAI Developer Platform, 访问于 2026 年 6 月. https://platform.openai.com/docs/api-reference/chat/create
[^2]: Anthropic, "Migrating to Claude 4," Anthropic Claude Docs, 访问于 2026 年 6 月, 以及 "Using the Messages API," https://docs.claude.com/en/api/messages-examples. https://docs.anthropic.com/en/docs/about-claude/models/migrating-to-claude-4
[^3]: Google, "GenerationConfig, Gemini API Reference," Google AI for Developers, 访问于 2026 年 6 月. https://ai.google.dev/api/generate-content#v1beta.GenerationConfig
[^4]: OpenAI, "Text generation" 与 prompt 工程指南, OpenAI Developer Platform, 访问于 2026 年 6 月. https://platform.openai.com/docs/guides/text-generation
[^5]: 关于按任务选 temperature 以及聊天“机器人腔”之争的实践者共识综述，汇总自多个二手来源，标注为共识。 https://tianpan.co/blog/2026-05-05-hyperparameter-illusion-temperature-top-p-last-to-tune
[^6]: Holtzman, A., Buys, J., Du, L., Forbes, M., Choi, Y., "The Curious Case of Neural Text Degeneration," ICLR 2020（提出核心采样）. https://arxiv.org/abs/1904.09751
[^7]: Yuan, J., Li, H., Ding, X., et al., "Understanding and Mitigating Numerical Sources of Nondeterminism in LLM Inference," arXiv:2506.09501, 2025 年 6 月提交, 2025 年 10 月修订. https://arxiv.org/abs/2506.09501
[^8]: Kindatechnical.com, "Why Temperature Zero Is Not Deterministic," 访问于 2026 年 6 月（二手、实践者来源）. https://kindatechnical.com/testing-non-deterministic-systems/why-temperature-zero-is-not-deterministic.html
[^9]: OpenAI, "Chat Completions API Reference, seed and system_fingerprint," OpenAI Developer Platform, 访问于 2026 年 6 月. https://platform.openai.com/docs/api-reference/chat/create
