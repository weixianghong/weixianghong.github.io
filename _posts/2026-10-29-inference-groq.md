---
title: "Groq"
date: 2026-10-29 09:02:00 -0700
permalink: /blogs/inference-groq/
lang: zh-CN
series: inference
series_order: 4
read_time: true
read_time_minutes: 4
excerpt: "芯片不做大，做很多颗，再让编译器提前排好计算和通信。Groq 的静态调度解决了什么，NVIDIA 为什么又把它和 GPU 拼起来？"
tags: [llm, inference, groq]
---

Cerebras 的办法是把芯片做得特别大。那另一条路呢？是芯片不做那么大，做很多颗，几百颗连起来。

Groq 就是这条路线。它和 Cerebras 的共同点，是把推理所需的权重放在片上 SRAM 里，尽量绕开反复从 HBM 搬权重的瓶颈。不同的是：**芯片之间怎么配合，在 Groq 这里从一开始就是主角。**

先分清两个名字：Groq 是做推理硬件和云服务的公司，不是 xAI 的 Grok 模型。

## 内存放近了，但芯片之间又远了

Groq 早期公开的 TSP 架构，每颗贡献 220 MiB 片上 SRAM，换成十进制大约是 230 MB。[^tsp] 一个几十 GB 的模型，显然不可能靠一颗装下，要分散到许多颗上。那要怎么让几百颗芯片配合好？

## 编译

GPU 不是毫无计划地运行：kernel、数据布局和算子调度都可以事先优化。但它也依赖运行时硬件调度、缓存和访存机制，不同执行单元会遇到不同的等待，通信也可能有争用。

Groq 更进一步，把大量工作交给编译器：**哪条指令什么时候执行，数据什么时候到某个单元，哪段时间用哪条链路，尽量事先排好。** 这通常叫静态调度、确定性执行，而不是等运行起来再由硬件处理所有细节。[^tsp]

它的 SRAM 也主要作为软件显式管理的工作存储，而不是让硬件缓存自己猜“下次会用什么”。新 LPX 仍沿用这个思路，并通过芯片间时钟漂移补偿、同步机制来维持协调。[^lpx]

<figure class="inference-figure">
{% include inference/static-schedule.svg %}
<figcaption>用三颗芯片举例：编译器同时规划计算与通信，让数据在下一步需要前到达。时间格只说明调度思路，不对应真实时钟数。</figcaption>
</figure>

这样做的价值，不是“所有芯片永远不用等”，而是把很多等待变成**已知、可以安排和重叠的时间**，减少不可预测的停顿。编译器有机会把这段等待藏在别的工作后面，几百颗如同像一颗那样跑。

代价也很清楚：编译器更复杂，模型和执行形状要能映射到这种时间表。把缓存和调度做得更通用，还是把执行安排得更确定，本来就是两种设计取舍。

## NVIDIA + Groq

2025 年 12 月 24 日，Groq 宣布与 NVIDIA 达成**非独占推理技术授权**；创始人 Jonathan Ross、总裁 Sunny Madra 等人加入 NVIDIA。不过，Groq 将继续独立运营，GroqCloud 也没有因此停掉。

2026 年 8 月 24 日，NVIDIA 宣布 Groq 3 LPX 已进入全面生产。它面向的是一个很具体的需求：不光一座机房一秒能产多少 token，还要让**每个人拿到 token 的速度够快，延迟够稳**。写代码、语音交互和多步 Agent，都会在意这件事。[^launch]

至于快多少，需要分开看。GroqCloud 的 gpt-oss-120b 文档目前列的是约 500 tokens/s。LPX 的一个新例子更明确：NVIDIA 在 2026 年 8 月公布，Artificial Analysis 用 **Gemma 4 31B、100K 输入上下文**测得中位输出速度 **3,431 tokens/s**；当时同图最快的公开服务是 870，约为四倍。数据快照是 8 月 20 日，公告是 8 月 24 日。[^bench]

## 更有意思的，是它怎么和 GPU 分工

SRAM 快，但容量还是紧。LPX 一柜 128 GB，里面除了权重，还要安排激活、KV 状态和运行时工作区。GPU 刚好有另一种长处：HBM 容量更大，生态也更通用。于是，NVIDIA 的混合推理方案里，有一种很自然的分工：

<figure class="inference-figure">
{% include inference/hybrid.svg %}
<figcaption>Attention–FFN 分离的一种配置：GPU 保留 KV Cache 并计算 attention，LPX 计算 FFN / MoE，往返传的是中间激活。不是每次把整段 KV Cache 搬过去。</figcaption>
</figure>

注意力需要读取长对话的 KV Cache，就放在 GPU 侧；FFN（前馈网络）或 MoE 专家层的权重，则常驻 LPU 的 SRAM。每层交换中间激活，让两边各做擅长的部分。

这和上一篇的 prefill–decode 分离还不一样：**前者按阶段切，Attention–FFN 分离是在一轮 decode 内按计算模块切。** 当然，NVIDIA 也公开了 prefill–decode 分离、外部草稿模型推测解码等其他组合。[^bench]

绕了一圈，问题又回到容量：把权重读取加速以后，KV Cache 的容量和读取成本就更显眼了。它到底存的是什么，为什么越聊越大，为什么又能让 API 便宜那么多？下一篇我们就来看看 KV Cache。

## Reference

[^tsp]: Groq，[ISCA 2022 论文](https://doi.org/10.1145/3470496.3527405)，分布式 SRAM、软件控制通信与确定性网络；220 MiB 是论文中的早期规格。
[^lpx]: NVIDIA，[Inside NVIDIA Groq 3 LPX](https://developer.nvidia.com/blog/inside-nvidia-groq-3-lpx-the-low-latency-inference-accelerator-for-the-nvidia-vera-rubin-platform/)，LP30 容量、带宽和编译器控制执行。
[^deal]: Groq，[非独占技术授权公告，2025-12-24](https://groq.com/newsroom/groq-and-nvidia-enter-non-exclusive-inference-technology-licensing-agreement-to-accelerate-ai-inference-at-global-scale)。
[^launch]: NVIDIA，[Groq 3 LPX 进入生产，2026-08-24](https://investor.nvidia.com/news/press-release-details/2026/NVIDIA-Groq-3-LPX-Now-in-Full-Production-With-World-Class-Speed-for-Agentic-AI/default.aspx)。
[^cloud]: Groq，[支持的模型及服务速度](https://console.groq.com/docs/models)。
[^bench]: NVIDIA，[Groq 3 LPX 的长上下文测试与混合推理配置](https://developer.nvidia.com/blog/how-nvidia-groq-3-lpx-unlocks-ultrafast-interactivity-at-long-context-on-nvidia-vera-rubin/)，包含测试时间、统一 tokenizer 及三种组合方式。
