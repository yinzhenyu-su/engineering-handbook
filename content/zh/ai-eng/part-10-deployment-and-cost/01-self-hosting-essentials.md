---
title: "自托管要点"
description: "要把开放权重模型跑得像样，你真正需要知道的东西：vLLM 作为默认服务端、作为资源约束的 KV cache、连续批处理，以及在掏钱之前如何估算 GPU 规模。"
slug: 01-self-hosting-essentials
part: 10
chapter: 1
difficulty: intermediate
prerequisites:
  - 00-hosted-vs-self-hosted
  - part-1-how-models-behave/01-tokens-and-context-windows
  - part-2-working-with-model-apis/00-calling-models
tags: [llm, deployment, cost]
date_created: 2026-06-12
date_updated: 2026-06-12
---

# 自托管要点

你手上有一个 Llama 3.3 70B 模型和两张 A100-80GB 显卡。老板想知道它能不能装得下。模型权重在 BF16 下是 140 GB，显然塞不进一张 80 GB 的卡。用张量并行（tensor parallelism）把权重摊到两张卡上，每张 70 GB。听起来很紧，听起来能跑。

其实跑不了，原因就是 KV cache。

本章是“把开放权重模型跑得像样”的操作清单。不讲 kernel 内部实现，不讲 GPU 架构。只讲你真正需要知道、用来配硬件、起服务、并且不让它崩掉的四件事：vLLM、KV cache、连续批处理，以及决定你租几张卡的量级估算。

## vLLM 是默认选择

在 NVIDIA GPU 上自托管开放权重模型，vLLM 就是答案。它最初在 UC Berkeley 开发，如今有 2000 多名贡献者，Meta、Stripe、Cohere、Mistral 都在生产环境里跑它。[^1] Stripe 在 2025 年 12 月把每天 5000 万次 API 调用从 HuggingFace Transformers 迁到 vLLM，用原先三分之一的 GPU 集群，报告推理成本下降 73%。[^1]

它成为默认选择，原因并不是激进的算子融合。在你能承担逐模型编译这一步时，TensorRT-LLM 在 NVIDIA 原生吞吐上比 vLLM 高 10-30%。vLLM 赢在硬件覆盖面（NVIDIA、AMD ROCm、AWS Trainium、Apple Silicon）、模型覆盖面（200 多个 HuggingFace 架构），以及它原生说 OpenAI API 这件事。

最后一点比听起来更重要：你的应用代码一行都不用改。

```python
from openai import OpenAI

# 同一个客户端，只换 base_url
client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

response = client.chat.completions.create(
    model="meta-llama/Llama-3.3-70B-Instruct",
    messages=[{"role": "user", "content": "Hello"}],
    max_tokens=256,
)
```

起服务只要一条命令：[^2]

```bash
# 单卡
vllm serve meta-llama/Llama-3.1-8B-Instruct

# 70B 模型的多卡张量并行
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 8192
```

三个参数干了大部分活。`--tensor-parallel-size` 把权重切到同一节点上的 N 张卡。`--gpu-memory-utilization`（默认 0.9）是 vLLM 可以拿去用的显存比例；设成 1.0 会在 CUDA graph 捕获阶段 OOM，所以别这么干。`--max-model-len` 限制上下文窗口。最后这个参数是整章里最重要的一个，接下来两节解释为什么。

## KV cache 是隐藏的成本

transformer 生成第 N+1 个 token 时，要对序列里所有之前的 token 算注意力。这些 token 的 key 和 value 会被缓存下来，免得每一步重算。没有缓存，生成成本按 token 数是 O(n^2)。有了缓存，每一步的开销大致恒定，而缓存本身线性增长。

精确的显存公式是本章唯一需要背下来的公式：[^3]

```text
KV_cache_bytes = 2 x L x B x T x H_kv x D_h x bytes_per_element
```

其中 `L` 是层数，`B` 是批次大小（batch size），`T` 是序列长度，`H_kv` 是 key-value 头数（分组查询注意力下远小于 query 头数），`D_h` 是头维度，`bytes_per_element` 在 BF16 下是 2、FP8 下是 1。

两个算例能让它落地。

**Llama 3.1 8B** 有 32 层、8 个 KV 头（GQA）、头维度 128。缓存每个 token 每个请求占 128 KB。4K 上下文、批次大小 1 时是 512 MB；批次大小 10 时大约 5 GB。单张 A100-80GB 绰绰有余。

**Llama 3.3 70B** 有 80 层、8 个 KV 头、头维度 128。缓存每个 token 占 320 KB。8K 上下文、批次大小 8 时是 20.5 GB。再加上 TP=2 时每卡 70 GB 的权重，你那两张 A100-80GB 就被钉死了。8K 上下文下，超过 8 个并发请求就没有余量，更别想用上模型原生的 128K 上下文窗口。

由此得出的决策规则是：`--max-model-len` 绝不要设得高于你实际请求长度的 95 分位。如果你的真实流量都是 4K 查询，模型那 128K 的能力毫无意义。

![一张柱状图，展示两种模型配置下 KV cache 显存随序列长度线性增长，较大模型的柱子把较小模型和权重线都压得很低，说明在长上下文下缓存本身就盖过了权重。](../../../ai-eng/part-10-deployment-and-cost/images/01-kv-cache-growth.png)
*KV cache 随上下文长度和批次大小线性增长。在 70B 这个量级、中等并发下，缓存比权重还吃显存。*

KV cache 还教给我们另一件事：GQA 是让 70B 模型真正可部署的那项架构创新。Llama 3.x 70B 用 8 个 KV 头，而不是 64 个 query 头。没有 GQA，同样 8K 上下文、8 个请求的批次，光 KV cache 就要 164 GB。这就是早年的 MHA 模型只能停在更小规模的原因。

给缓存付账有几种方式：

- **调低 `--max-model-len`**（最便宜，不损失质量，只是上下文变短）
- **用 `--kv-cache-dtype fp8_e5m2`**（缓存显存减半；多数负载质量影响极小，数学和代码上略差）
- **量化权重**，用 `--quantization fp8`（把权重占的显存让给缓存）
- **加卡**，通过 `--tensor-parallel-size`（最贵的那一种）

## 连续批处理是吞吐量的诀窍

vLLM 之所以存在，靠的是一个叫连续批处理（continuous batching）的调度技巧。朴素做法（静态批处理）把 N 个请求凑成一批一起跑，等到最长那个结束才收新请求。如果一个用户要一篇 2000 token 的长文，另外七个人只要 50 token 的回答，那七个短请求的槽位就被锁住，白白多跑 1950 个解码步。

连续批处理在 OSDI 2022 的 Orca 论文里以迭代级调度（iteration-level scheduling）的名字提出，它在每个解码步重新计算活跃请求集合。[^4] 一个请求打到 EOS token，槽位立刻释放，排队中的请求补上。没人再等最长的那个请求了。

吞吐数字并不含糊。在 GPT-3 175B 上、延迟对齐的条件下，Orca 测到比 FasterTransformer 高 36.9 倍的吞吐。[^4] vLLM 2023 年最早的基准显示，视并发不同，比 HuggingFace Transformers 快 2-24 倍。放到 Stripe 的规模上，这就等于三分之一的 GPU 集群。[^1]

在 vLLM 里没有开关来控制它，它一直开着。你真正要调的是 `max_num_seqs`，它限制运行队列里的并发序列数。设太高，KV 缓存池耗尽，vLLM 开始驱逐请求，等容量回落后这些请求要从头重跑 prefill，你的长尾延迟就爆了。设太低，GPU 就闲在那里。

正确的调法是盯 `vllm_gpu_cache_usage_perc` 这个 Prometheus 指标。如果它一直低于 70%，就调高 `max_num_seqs`。如果它顶在 90% 且 `vllm_num_preempted_requests` 在涨，就调低 `max_num_seqs` 或者加卡。

还有一件事值得知道：vLLM 也支持*分块预填充*（chunked prefill），它把长提示切成块，让一次 32K token 的 prefill 不至于把整步的解码请求全堵住。V1 里默认开启。你平时不会想到它，直到你发现延迟尖峰和长提示的到达时间对得上，那就把 `long_prefill_token_threshold` 调小一点。

## 不用计算器也能估算 GPU 规模

在租硬件之前，先花九十秒做量级估算。三步。

**第一步：权重显存。** `weight_GB = params_billions x bytes_per_param / TP`，其中 bytes_per_param 在 BF16 下是 2、FP8 下是 1、INT4 下是 0.5。Llama 3.3 70B 用 BF16、TP=4，就是 70 x 2 / 4 = 35 GB 每卡。同款模型用 FP8、TP=2，也是 70 x 1 / 2 = 35 GB 每卡。

**第二步：KV 缓存预算。** 也就是权重和开销之后剩下的部分。默认设置下 vLLM 的显存布局：[^5]

```text
GPU 总显存
  - 预留预算 = 总量 x gpu_memory_utilization（默认 0.9）
      - 模型权重
      - 激活值 + NCCL 缓冲（约 2-3 GB）
      - KV 缓存池（其余全部）
  - 未预留的 10%
      - CUDA graph（1-5 GB）
```

所以单张 80 GB A100、TP=2 跑 Llama 3.3 70B 时：预算是 72 GB，权重 35 GB，开销约 2 GB，每卡 KV 池约 35 GB。两张卡合起来就是 70 GB 的缓存池。空间很充裕。

**第三步：核对目标并发。** 用前面那个公式：

```python
def kv_cache_gb(num_layers, num_kv_heads, head_dim, max_seq_len, batch_size, dtype_bytes=2):
    """Exact KV cache memory in GB across the full batch."""
    total_bytes = (
        2 * num_layers * batch_size * max_seq_len * num_kv_heads * head_dim * dtype_bytes
    )
    return total_bytes / (1024 ** 3)

# Llama 3.3 70B：80 层、8 个 kv_heads、head_dim 128、batch=16、8K token
kv_needed = kv_cache_gb(80, 8, 128, 8192, 16)
# 需要 40 GB，可用约 70 GB -> 装得下，还有富余
```

如果 `kv_needed` 超过了你的 KV 池：调低 `--max-model-len`、调低 `max_num_seqs`、换 FP8 KV cache，或者加卡。没有免费的午餐，但顺序是：先降上下文，其次 FP8 缓存，最后加卡。

一个模型所需的最少卡数，取决于权重加上一个像样的 KV 池能装进单卡的最小张量并行规模。然后向上取到 2 的幂。

## 最先咬人的几个坑

头一个月跑 vLLM，有六种故障会咬到你。它们都有成熟的修法，你只需要认出它们。

**启动就 OOM。** 默认上下文长度对可用显存来说太长。加上 `--max-model-len 8192`（或者更小）。报错信息里常常会给出一个安全值。

**请求被静默抢占。** 吞吐看着正常，但 p99 延迟烂透了。KV 缓存池耗尽，vLLM 正在中途驱逐请求。盯 `vllm_num_preempted_requests`。只要它不是 0，就调低 `max_num_seqs` 或者加容量。

**张量并行跑在 PCIe 上。** 你的 TP=4 部署居然比单卡还慢。张量并行在每次前向都要做一次 all-reduce。NVLink 给到约 600 GB/s，PCIe Gen4 只有约 64 GB/s。光 all-reduce 就能吃掉 PCIe 上 20-40% 的单步时间。用 NVLink 互联的卡，做不到就换流水线并行。

**首个请求的冷延迟。** 启动后的第一个请求比后面那些慢 60 倍。CUDA graph 还在预热，权重也还没进 L2 缓存。在负载均衡器把流量导到新实例之前，先在健康检查里打几个预热请求。

**上下文窗口对不上。** 用户发来 16K token，收到 400 错误。模型的 `config.json` 里写着 `max_position_embeddings=4096`，哪怕 RoPE 缩放能支持更长。把 `--max-model-len` 显式设成你真正测过的长度。

**TGI 已经死了。** HuggingFace 在 2025 年 12 月 11 日把 Text Generation Inference 转入维护模式。别拿它起新部署。通用负载用 vLLM，前缀密集的 RAG 和结构化输出用 SGLang。

这就是操作清单。下一章是定制清单（微调、LoRA、蒸馏），再往后是[成本工程](03-cost-engineering.md)，本章的每一个旋钮都会在那里重新出现，变成影响你月度账单的杠杆。

## 参考文献

[^1]: Introl, "vLLM Production Deployment", December 2025, https://introl.com/blog/vllm-production-deployment-inference-serving-architecture
[^2]: vLLM, "Quickstart", official documentation, June 2026, https://docs.vllm.ai/en/latest/getting_started/quickstart/
[^3]: Michael Brenndoerfer, "KV Cache Memory: Calculating GPU Requirements for LLM Inference", January 2026, https://mbrenndoerfer.com/writing/kv-cache-memory-calculation-llm-inference-gpu
[^4]: Gyeong-In Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models", USENIX OSDI 2022, https://www.usenix.org/conference/osdi22/presentation/yu
[^5]: NVIDIA, "Troubleshooting GPU Memory Out-of-Memory Errors", NVIDIA NIM for LLMs v2.0.5 docs, June 2026, https://docs.nvidia.com/nim/large-language-models/2.0.5/troubleshooting/memory.html
