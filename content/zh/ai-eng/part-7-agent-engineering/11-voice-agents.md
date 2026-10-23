---
title: "语音智能体"
description: "定义语音智能体的 800 毫秒预算、级联与原生之间的权衡，以及为什么打断是整条链路里最难的正确性问题。"
slug: 11-voice-agents
part: 7
chapter: 11
difficulty: intermediate
prerequisites:
  - part-2-working-with-model-apis/01-streaming
  - 02-the-agent-loop
  - 03-tool-design
tags: [llm, agents]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 语音智能体

在对话分析研究者研究过的每一种语言的对话里，一个人说完一句话到另一个人开口，间隔中位数大约是 200 毫秒。拖到 500 毫秒以上，听的人就认为对方在思考。拖到 800 毫秒以上，就会被判定为坏了，或者像个机器人。[^1] 语音智能体（voice agent）之所以难，全部原因就是这个数字。端到端不到 800 毫秒，你要完成：判断用户已经停止说话、把说的话转写成文字、跑一遍 LLM、合成语音、然后开始播放音频。一旦超了，这通对话就是感觉不对——用户说不清哪里不对，但不会忍。

本章的每一项内容——架构选型、端点检测的技巧、打断协议、传输方式——都是在往这个预算里挤。

## 800 毫秒去哪了

从用户开口到播出第一帧音频，这条链路大约拆成六个阶段。下面是 2025-2026 年的生产环境数据，p50 和 p95。[^2]

| 阶段 | 这一步在做什么 | p50 | p95 |
|---|---|---|---|
| 轮次结束检测 | 判断用户已经说完 | 200-300 ms | 400 ms |
| 最终 ASR 转写 | STT 定下最后几个词 | 50-150 ms | 250 ms |
| LLM 首 token 时间 | 提示词拼装完毕，模型开始输出 | 200-400 ms | 700 ms |
| TTS 首块音频 | 从首个 token 到听得见的声音 | 100-200 ms | 350 ms |
| 网络抖动 | WebRTC 或 PSTN 传输 | 30-80 ms | 150 ms |
| **总计** | | **~600-850 ms** | **~1,450 ms** |

这个总计只有在每一段都流式、并且各段相互重叠时才装得下。TTS 不等 LLM 说完，后面的 token 还在解码，它已经开始合成第一句。播放器也不等 TTS 输出完整，音频块到一块就播一块。如果流水线里每一段都阻塞下一段，用户还没开口，预算就已经超了两三倍。这不是一项优化，这就是架构本身。

![一张横向条形图，含两条堆叠的条形。上方条形标注为「Streaming, overlapping」，长度约 800 毫秒，被分成五段彩色区段，分别标注 End-of-turn、ASR、LLM TTFT、TTS first、Network。下方条形标注为「Sequential, blocking」，长度约 2500 毫秒，同样的五段首尾相接、互不重叠。一个珊瑚色箭头指向两个总计之间的差值。](../../../ai-eng/part-7-agent-engineering/images/11-latency-budget.png)
*同样的五个阶段，两种架构。流式加重叠能装进 800 毫秒；顺序阻塞超了 3 倍。*

## 两种架构，一个权衡

上面的流水线有两种搭法。每一种都放弃了另一种保留的东西。

**级联式流水线（cascaded pipeline）** 把若干独立服务串起来。语音活动检测（VAD）判断用户什么时候停止说话，流式语音识别（STT）模型做转写，LLM 生成文本，流式语音合成（TTS）模型合成音频，最后播放出来。Pipecat（Daily，2024 年 5 月）和 LiveKit Agents 这类框架是生产标准实现。[^3] 每一段都是独立进程，可以单独替换、单独记日志、单独测试。你可以在 STT 到 LLM 的交界处读到完整转写文本，在那里跑护栏、注入[检索](../part-6-retrieval-augmented-generation/00-when-you-actually-need-rag.md)、执行合规规则，全都是纯文本。

**原生语音到语音（speech-to-speech，S2S）** 路线把原始音频送进一个多模态模型，模型直接吐回原始音频。OpenAI 的 `gpt-realtime-2`（2025 年正式可用）是生产参考实现。[^4] 中间不存在任何文本。模型能用上声学线索（语气、犹豫、情绪），而这些线索在音频变成转写文本的那一刻就消失了。STT 输出和 TTS 输入这两次序列化跳转没有了，延迟下界就是从这儿降下去的。

权衡很直接。级联给你可观测性和控制力；原生给你更低的延迟下界和声学感知。医疗、金融、客服领域的多数生产部署选级联，因为合规团队需要每一轮的文本日志，而他们的 RAG 层必须放在 STT 和 LLM 之间。做消费级语音应用的团队，如果延迟本身就是产品，就选原生。[^5]

```text
级联：      [VAD] -> [STT] -> [文本边界] -> [LLM] -> [TTS]
                                    ^
                                    在这里记日志、在这里检索、在这里跑护栏

原生 S2S：  audio -----> [多模态模型] -----> audio
                              （没有文本边界）
```

默认做法：从级联开始。只有当延迟是硬约束、并且你能接受没有中间文本时，才升级到原生。一条调好的级联流水线，服务同机部署、STT 中间结果在 100 毫秒以内，对大多数负载来说已经足够接近原生，用户根本察觉不到差别。[^6]

## 端点检测：抢话与冷场的两难

轮次结束检测（endpointing，端点检测）是整条流水线里杠杆最大的那个旋钮。设错了，要么模型还没见到一个 token，你就先花掉了一半延迟预算；要么用户正念着信用卡号念到一半，被你打断。

主流做法有三种。

- **基于静音的 VAD。** 音频能量持续低于阈值一段时间就触发，默认 500 毫秒。快且便宜，但用户一停下来思考就被打断。在 OpenAI 的 Realtime API 里，这就是 `turn_detection.type = "server_vad"`，可调 `threshold`、`prefix_padding_ms` 和 `silence_duration_ms`。[^7]
- **语义端点检测。** 一个小分类器给音频或转写文本打分，判断这一轮是不是真的说完了。OpenAI 把它暴露成 `type: "semantic_vad"`，带一个 `eagerness` 旋钮（`low`、`medium`、`high`）。低 eagerness 让用户把复杂的想法说完；高 eagerness 切得很激进。AssemblyAI 的 Universal-Streaming 用混合方案：转写过程中预测一个轮次结束 token，配一个置信度阈值（默认 0.7），置信时最少静音 160 毫秒，边界情况回落到 2,400 毫秒。[^8]
- **纯文本语义模型。** LiveKit 开源了 `livekit/turn-detector`，把 Qwen2.5-0.5B 蒸馏成 INT8 ONNX，对最近 6 轮的转写文本分类，给出话语结束的概率。支持十四种语言，主要语种的真阳性率超过 99%，纯 CPU 运行，内存不到 500 MB。[^9]

几乎所有人最先犯的错，是在采集实体信息时继续用基于静音的 VAD。用户开始念 16 位卡号，在数字分组之间自然停顿，智能体念完前四位就触发了。在这些流程上换成语义端点检测之后，LiveKit 在结构化数据上的失败率相对下降了 39.23%。[^9] 但别到处都用。快问快答的场景里，语义端点检测的谨慎显得迟钝，短阈值的静音方案更好。

正确的规则是按流程分：采集实体和需要耐心倾听的段落，用低 eagerness 的语义端点检测；要快速来回的场景，用基于静音或高 eagerness 的语义方案。

## 打断（barge-in）：截断到用户真正听到的位置

这一块在文本智能体里没有对应物，每个上线语音产品的团队都被它咬过。智能体正说到一半，用户开口了。三件事必须原子地、按正确顺序发生：

1. 立刻停止 TTS 播放。
2. 取消正在进行的 LLM 生成。
3. **把对话历史截断到用户实际听到的位置，而不是已经生成的位置。**

第三条是陷阱。如果智能体说的是「Your balance is $500 and your next payment is due on the 15th」，用户在「and your next」处打断，那么模型下一轮看到的助手消息必须以「and your next」结尾，后面什么都没有。要是把整条消息留在历史里，下一次回复就会提到一个用户从没听过的还款日期；表现出来就是用户说「what? you said something about a date?」，而智能体回一句驴唇不对马嘴的话。

OpenAI 的 Realtime API 按传输方式不同，处理方式也不同。走 WebRTC 时，输出缓冲区在服务端，服务端知道已经播了什么，收到 `input_audio_buffer.speech_started` 就自动截断。[^10] 走 WebSocket 时，播放由客户端负责，这些活得自己干：以毫秒为单位记录已播放的音频时长，用户开口时冻结这个计数，然后带着该时间戳发一个 `conversation.item.truncate` 事件。

```python
import json

def handle_barge_in(ws, last_response_item_id: str, played_ms: int) -> None:
    """Stop playback, then truncate history to what was heard."""
    truncate_event = {
        "type": "conversation.item.truncate",
        "item_id": last_response_item_id,
        "content_index": 0,
        "audio_end_ms": played_ms,
    }
    ws.send(json.dumps(truncate_event))
    # 服务端发出 response.cancelled；新的用户轮次开始。

def on_server_event(ws, message: str, playback_state: dict) -> None:
    event = json.loads(message)
    if event["type"] == "input_audio_buffer.speech_started":
        item_id = playback_state.get("current_item_id")
        played_ms = playback_state.get("played_ms", 0)
        if item_id:
            handle_barge_in(ws, item_id, played_ms)
```

真正坑人的是 `played_ms`。它是音频播放时间，不是墙上时钟时间。用一个单调计数器，只在播放器真正消费掉一个音频块时才往前加。按墙上时钟量，会漏掉那些进了缓冲区却从没到扬声器的音频。

OpenAI 的文档明确提到一个细节：「the realtime model doesn't have enough information to precisely align transcript and audio.」截断是干净的一刀，不是半个句子。你不会拿回一段没播出去的半截词碎片；拿到的是裁到最后一个完整播出 token 的转写文本。[^10]

另一种失效模式是**误打断（false barge-in）**。背景噪声、麦克风回声，或者用户一句「uh-huh」的应和，都会触发 VAD，智能体说到一半就停了。Hamming 分析了 10,000 个语音智能体、400 万通生产通话，发现它稳稳排在前五类失效模式里。[^11] 修法要分层：在 WebRTC 传输层开启回声消除（浏览器默认开，服务端要自己配）；嘈杂环境把 VAD 的 `threshold` 从 0.5 提到 0.7-0.8；再加一个应和过滤器，把三个词以下的短句造成的打断压掉。

还有一条让人意外的规则：打断不该是一个布尔开关。法律告知、支付确认、合规要求的固定话术，设计上就必须不可被打断。这是对话管理器里按消息类型配置的策略，不是 VAD 配置里的一个开关。[^11]

## 传输层：用 WebRTC，不用 HTTP

HTTP 是请求-响应模式。语音是全双工：音频同时双向流动，连接在整通对话里一直开着，传输层还得处理抖动、丢包和回声。这几件事 HTTP 一件也做不好。WebRTC 就是为这个任务设计的：ICE 做 NAT 穿透，DTLS+SRTP 加密媒体流，Opus 编解码器带自适应码率，浏览器侧做回声消除和抖动缓冲，RTCP 反馈质量。[^12]

分工很简单：

- **WebRTC** 用于浏览器和移动客户端。回声消除和抖动缓冲浏览器替你做了。用 OpenAI 的 Realtime API 走 WebRTC 时，服务端还会自动处理打断截断。[^4]
- **WebSocket** 用于服务端已经从媒体流水线（通常是电话系统）拿到原始音频的场景。截断要自己实现，就像上面那段代码那样。[^10]
- **SIP** 用于通过 Twilio、Telnyx、Plivo 这类服务商直连电话系统。PSTN 在 G.711 上再加 20-50 毫秒单向延迟，运营商抖动再占 30-80 毫秒，这些总得从你那 800 毫秒里出。[^2]

把 WebRTC 扩到生产规模，本身就是一道工程难题。OpenAI 2026 年 5 月的基础设施博客讲了他们怎么把 Realtime 拆成一个无状态的 Go 中继（按 ICE ufrag 路由提示做 UDP 转发）和一个有状态的收发器（真正承载 ICE/DTLS/SRTP 会话）。拆开之后就能跑在 Kubernetes 里，不用像原生 WebRTC 那样把每个会话的 UDP 端口段暴露出去；他们靠这个给 9 亿周活跃用户提供语音服务。[^13]

## 填充式应答

当 LLM 首 token 时间预计超过 400 毫秒左右——通常是因为要调一次 CRM 或知识库的工具——用户会听到静音，开始怀疑是不是断线了。解法是一句填充话术：一句预先录好或预合成的短语（「Sure, let me check...」或「One moment...」），工具调用一开始就播。这买到 1-2 秒的体感响应，实际延迟一点没变。

两条规则。填充话术不要写进对话历史，播完就扔。也不要每一轮都用，只在确实预计会卡的地方用。用过头，破坏自然度的速度比它本来要遮的那段静音还快。

## 语音专属的评测

文本评测覆盖不到语音。真正重要的指标不一样，其中有几项是纯文本系统永远看不到的维度。

- **ASR 质量。** 目标口音和噪声条件下的词错率（WER），再加上流式专属的指标：首个中间转写出现的时间、从说完到最终转写的时间，以及不可变性（服务商会不会回头改写已经确定的词？）。AssemblyAI 的 Universal-Streaming 输出不可变转写；有些服务商不是。[^8]
- **端点检测质量。** 两种失效模式分开打分：假阳性率（把用户打断）和假阴性率或延迟（冷场）。Artificial Analysis 的 AA-WER Streaming 基准把 WER 和转写耗时一起报，这个框架是对的。[^14]
- **轮次级延迟。** p50 和 p95 下的语音到首帧音频，端到端从说完话量到第一个听得见的字节。这是首要的体验指标。
- **打断指标。** 打断率、误打断率（没有有效转写文本的打断），以及打断之后的任务完成率。Hamming 的结论是最该记住的那句：「the dashboard says p95 turn latency is healthy, but callers hear the agent cut them off mid-account-number.」聚合延迟会把打断失败藏起来。[^11]
- **端到端任务完成率。** 把合成音频直接灌进流水线（不用麦克风），抓取输出，用自动指标加 LLM 裁判给任务是否成功打分。Amazon 的 Nova Sonic 团队公开过这套规模化语音评测方法；要跑几千通测试通话，这是唯一现实的做法。[^15]

更底层的机制（LLM 裁判、回归套件、线上评测）在[第 8 部分](../part-8-evaluation-and-observability/00-the-eval-taxonomy.md)。这里要说的是：那些文本评测技术，不补上上面这四项指标，一样搬不到语音上。

## 什么会在生产环境里杀死语音智能体

这五种失效模式值得练到肌肉记忆里：

1. **顺序流水线把预算花超。** 各段用阻塞式 HTTP 调用实现，没有流式。首帧音频延迟中位数超过 2 秒，ASR 最终结果到 TTS 首帧之间有大段空档。修法：每个边界都流式。
2. **打断后历史不一致。** `audio_end_ms` 传错（用了墙上时钟而不是播放时间），或者压根没截断。修法：单调播放计数器，每次打断都发 `conversation.item.truncate`。
3. **回声或应和造成的误打断。** 背景噪声一来，智能体就说到一半停了。修法：开 AEC、提高 VAD 阈值、过滤短的应和。
4. **实体口述时过早端点检测。** 基于静音的 VAD 在卡号中间的自然停顿上触发了。修法：按流程配置端点检测策略，采集时用低 eagerness 的语义方案。
5. **长会话延迟漂移。** 对话历史越积越多，TTFT 从 300 毫秒爬到 800 毫秒以上。这就是[智能体循环那一章](02-the-agent-loop.md)讲的上下文窗口问题。修法：轮换会话，或者每 10-15 分钟摘要一次历史。[^16]

800 毫秒的预算能达到，但不是白给的。托管实时 API 的美国区首字节时间本身就要 500 毫秒左右，留给采集、VAD、网络和渲染的只剩大约 300 毫秒——而这还是在你听到一个字之前。本章每一个架构选择，都是从那 500 毫秒的下界走到一个能装进人类对话节奏的系统所要付的代价。[^16]

## 参考文献

[^1]: Stivers et al., "Universals and cultural variation in turn-taking in conversation," PNAS, 2009; Levinson and Torreira, "Timing in turn-taking and its implications for processing models of language," Frontiers in Psychology, 2015. Cited via Twig AI, "Latency Budgets for Voice AI Agents: The 800ms Rule of Natural Conversation," May 2026, https://www.twig.so/blog/voice-ai-agents-latency-budget-800ms

[^2]: Chandan Maruthi, "Latency Budgets for Voice AI Agents: The 800ms Rule of Natural Conversation," Twig AI Blog, May 2026, https://www.twig.so/blog/voice-ai-agents-latency-budget-800ms

[^3]: Pipecat / Daily.co, "Overview of Pipecat," Pipecat Docs, 2025, https://docs.pipecat.ai/guides/learn/overview

[^4]: OpenAI, "Realtime and audio - overview," OpenAI Developer Docs, June 2026, https://platform.openai.com/docs/guides/realtime

[^5]: deepsense.ai, "Realtime Voice AI in the Enterprise: Overcoming Latency with Native Audio Models," 2026, https://deepsense.ai/blog/realtime-voice-ai-in-the-enterprise-overcoming-latency-with-native-audio-models/

[^6]: forasoft.com, "Production Voice Agents (2026)," 2026, https://www.forasoft.com/blog/article/openai-realtime-api-voice-agent-production-guide-2026

[^7]: OpenAI, "Voice activity detection (VAD)," OpenAI Developer Docs, June 2026, https://platform.openai.com/docs/guides/realtime-vad

[^8]: Martin Schweiger, "How intelligent turn detection (endpointing) solves the biggest challenge in voice agent development," AssemblyAI Blog, August 28, 2025, https://assemblyai.com/blog/turn-detection-endpointing-voice-agent

[^9]: LiveKit, "livekit/turn-detector model card," Hugging Face, 2025, https://huggingface.co/livekit/turn-detector/blob/main/README.md

[^10]: OpenAI, "Realtime conversations - Interruption and Truncation," OpenAI Developer Docs, June 2026, https://platform.openai.com/docs/guides/realtime-conversations

[^11]: Sumanyu Sharma, "Voice Agent Interruption Handling: Barge-In, Backchannels, and Turn Detection," Hamming AI, May 2026, https://hamming.ai/blog/voice-agent-interruption-handling-runbook

[^12]: W3C/IETF, "WebRTC 1.0: Real-Time Communication Between Browsers," accessed June 2026, https://www.w3.org/TR/webrtc/

[^13]: Yi Zhang and William McDonald, "How OpenAI delivers low-latency voice AI at scale," OpenAI Engineering Blog, May 4, 2026, https://openai.com/index/delivering-low-latency-voice-ai-at-scale

[^14]: Artificial Analysis, "New Speech to Text Streaming Benchmark (AA-WER Streaming)," 2025, https://artificialanalysis.ai/articles/new-streaming-speech-to-text-benchmark-aa-wer-streaming

[^15]: AWS / Amazon, "Evaluate your Amazon Nova Sonic voice agent at scale, no microphone required," AWS Machine Learning Blog, May 2026, https://aws.amazon.com/blogs/machine-learning/evaluate-your-amazon-nova-sonic-voice-agent-at-scale-no-microphone-required/

[^16]: forasoft.com, "Production Voice Agents (2026)," 2026, https://www.forasoft.com/blog/article/openai-realtime-api-voice-agent-production-guide-2026
