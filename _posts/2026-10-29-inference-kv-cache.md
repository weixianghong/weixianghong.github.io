---
title: "KV Cache"
date: 2026-10-29 09:03:00 -0700
permalink: /blogs/inference-kv-cache/
lang: zh-CN
series: inference
series_order: 5
read_time: true
read_time_minutes: 8
excerpt: "同样一百万输入 token，缓存命中和未命中的价格可以差 50 倍。从 KV Cache 怎么复用、为什么占显存，一直讲到 Agent 怎么管理上下文。"
tags: [llm, inference, kv-cache, agent]
---

同样 100 万个输入 token，交给 DeepSeek 的 API，未命中缓存收 1 块钱，命中只收 2 分钱，差了 50 倍。[^price]

## 前缀不变，对应的 KV 就能复用

回到系列第一篇的 QKV。在某一层里，新位置产生自己的 Q、K、V，用 Q 去读取前面位置的 K、V。前面那些位置的 K、V 如果已经算过，就把它们留下，下次直接用。

这里说的是 **causal attention 下的自回归推理**，不是统计学里的 causal inference。对通常的 Transformer 来说，在模型、positional encoding 和其他输入条件都不变时，一个位置的表示只依赖它及前面的 token，不依赖后面还没出现的 token。

因此，**前缀不变，对应的 KV 就能复用。** [^cache]

<figure class="inference-figure">
{% include inference/kv-growth.svg %}
<figcaption>每一层保留旧位置的 K、V，新增位置只追加自己的 K、V。Q 负责本步读取，通常不需要像历史 K、V 那样缓存。这里画的是全局注意力的简化过程。</figcaption>
</figure>

这样，生成下一个 token 时，不必重新给整个前缀跑一遍模型。

很多 chat 或 agent 的调用，看起来就是一个消息列表：system、user、assistant，再接 user、assistant。Agent 还会穿插工具定义、tool 消息和工具返回的结果。

于是前缀几乎和上次一样，上一次留下来的 KV，这一次就可能继续用。这通常叫 **prefix caching / prompt caching**。[^prefix]

命中时，匹配前缀的 prefill 可以省掉；只需要算新追加的部分即可。

<figure class="inference-figure">
{% include inference/prefix.svg %}
<figcaption>缓存按 token 前缀匹配，不是按语义相似度匹配。在中间改了一块，即使后面的文字相同，后面的普通全局注意力缓存也不能直接接着用。</figcaption>
</figure>

## KV Cache

KV Cache 的大小，不能只看模型一共有多少参数。对标准全局注意力，常用的粗估是：

$$
M_{\rm KV}=2\times L\times T\times H_{\rm KV}\times D\times s.
$$

2 是 K 和 V，$L$ 是层数，$T$ 是 token 数，$H_{\rm KV}$ 是 KV head 数，$D$ 是每个 head 的维度，$s$ 是每个元素的字节数。多条互不共享前缀的请求，还得分别占空间。

拿 gpt-oss-120b 来算，它有 36 层、8 个 KV head，head dimension 是 64。假设 KV 用 BF16，每元素 2 字节；每层每个位置需要：

$$
2\times8\times64\times2=2048\text{ bytes}=2\text{ KiB}.
$$

但它的 **18 层是全局注意力，18 层是 128-token 滑动窗口**。窗口层不必一直保存整个长历史。如果实现确实按窗口裁剪缓存，131,072 个位置大约需要：[^config]

$$
\begin{aligned}
M_{\rm KV}
&\approx18\times131072\times2\text{ KiB}\\
&\quad+18\times128\times2\text{ KiB}\\
&\approx4.50\text{ GiB}\approx4.84\text{ GB}.
\end{aligned}
$$

可以粗略的认为每个对话的 KV Cache 大约5 GB。再加上几十 GB 的量化权重，一张 80 GB H100 留给长对话的余量就不多了。

## KV Cache 多级保存

decode 时反复读取的 KV，适合留在 HBM 这样的高速内存里。否则每一步都从 CPU 内存或 SSD 往回搬，省下容量，却可能赔掉延迟。[^cache]

一次调用结束后，如果下一次命中的概率较低，就不一定还占着宝贵的 HBM，可以降到 CPU 内存或 SSD，命中时再恢复到参与计算的内存里，后续 decode 才能继续快速读取。DeepSeek 就公开介绍过磁盘上下文缓存。[^disk]

这也解释了为什么缓存不是想留多久就留多久：存储要钱，恢复要带宽，系统还要腾空间给新请求。于是有过期和淘汰；有的平台还对缓存写入单独收费。比如 Anthropic 的 5 分钟、1 小时写入采用不同费率。[^billing]

## Context Engineering & Harness Engineering

把几个常见名字放到一起，就比较好记了：

<div class="inference-table" markdown="1">

| 办法 | 主要在改什么 |
|---|---|
| GQA | 多个 Q head 共享一组 KV head，减少要保存的 KV |
| MLA | 用潜在表示压缩 KV 状态，改变注意力结构 |
| KV 量化 | 每个元素少用一些 bit，用精度与额外处理换容量 |
| vLLM 的 PagedAttention | 把缓存分块管理，减少碎片和重复占用 |
| SGLang 的 RadixAttention | 找到共同前缀，更方便地跨调用共享缓存 |

</div>

它们不是同一种优化：有的改模型结构，有的改数值格式，有的改内存管理和复用方式。PagedAttention 也不是把模型数学上需要的 KV 突然压没了。[^optimizations]

## Agent

由于 Agent 会一遍遍带着旧上下文继续调用模型，所以它们是 KV Cache 受益的大户。OpenRouter 在 **2026 年 10 月 8 日**发布的报告里，统计了 9 月流量：按它对 agentic API key 的分类，缓存输入 token 接近全部 agentic token 的 90%。[^router]

因此，context engineering、harness engineering 很大一部分功夫，就是在管理这个上下文：放什么，按什么顺序，工具结果留多久，什么时候压缩。

这里有个冲突。内容决定模型做得好不好，前缀稳不稳影响缓存能不能复用。比如把前面一段没用的内容删掉，后面的 token 位置和上下文都变了，普通前缀缓存从这里开始就接不上了，要重新 prefill。

一般来说，稳定的指令和工具定义尽量放前面，顺序保持一致；新的任务信息和工具结果往后追加。压缩不必每步都做，可以等上下文的成本、长度或质量真的值得时，再接受一次缓存重建。

但也别为了命中率，把明明没用甚至误导模型的东西一直留着。省了这一轮 prefill，后面每个 token 的 attention 可能更贵，任务还可能做错。

**缓存命中率是 harness 很重要的成本和延迟指标，但不是它好不好的唯一标准。** 我会把它和任务成功率、总费用、端到端耗时放在一起看。一个很会复用缓存、却把事情做错的 Agent，没有什么好高兴的。

## Reference

[^price]: DeepSeek，[模型与价格](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/)，Flash 空闲时段人民币输入价。
[^cache]: Hugging Face，[Cache strategies](https://huggingface.co/docs/transformers/kv_cache)，滑动窗口、offload 与量化缓存。
[^prefix]: DeepSeek，[Context Caching](https://api-docs.deepseek.com/guides/kv_cache/)，前缀持久化、命中规则及 best-effort 行为。
[^config]: OpenAI，[gpt-oss-120b 配置](https://huggingface.co/openai/gpt-oss-120b/blob/main/config.json)与[参考实现](https://github.com/openai/gpt-oss/blob/main/gpt_oss/torch/model.py)。缓存大小由配置自行推算，假设窗口层只保留窗口状态。
[^disk]: DeepSeek，[Context Caching on Disk](https://api-docs.deepseek.com/news/news0802/)，此处引用其机制，历史价格不作为现价。
[^billing]: Anthropic，[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)，TTL 与缓存写入计费。
[^optimizations]: [GQA 论文](https://aclanthology.org/2023.emnlp-main.298/)；[DeepSeek-V2 / MLA 论文](https://arxiv.org/abs/2405.04434)；[vLLM：PagedAttention](https://vllm.ai/blog/2023-06-20-vllm)；[LMSYS：RadixAttention 与 SGLang](https://www.lmsys.org/blog/2024-01-17-sglang/)。
[^router]: OpenRouter，[Cache is Crucial](https://openrouter.ai/data/newsletter/cache-is-crucial)，2026-10-08 发布，统计 9 月数据。
