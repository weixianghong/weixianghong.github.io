---
title: "Groq"
date: 2026-10-29 09:02:00 -0700
permalink: /blogs/inference-groq/
lang: zh-CN
series: inference
series_order: 4
read_time: true
read_time_minutes: 6
excerpt: "芯片不做大，做很多颗，再让编译器提前排好计算和通信。Groq 的静态调度解决了什么，NVIDIA 为什么又把它和 GPU 拼起来？"
tags: [llm, inference, groq]
---

Cerebras 的办法是把芯片做得特别大。那另一条路呢？芯片不做那么大，做很多颗，几百颗连起来。

Groq 就是这条路线。它和 Cerebras 的共同点，是把推理所需的权重放在片上 SRAM 里，尽量绕开反复从 HBM 搬权重的瓶颈。不同的是：**芯片之间怎么配合，在 Groq 这里从一开始就是主角。**

先分清两个名字：Groq 是做推理硬件和云服务的公司，不是 xAI 的 Grok 模型。

## 内存放近了，芯片之间又远了

Groq 早期公开的 TSP 架构，每颗贡献 220 MiB 片上 SRAM，换成十进制大约是 230 MB。[^tsp] 一个几十 GB 的模型，显然不可能靠一颗装下，要分散到许多颗上。

而现在的 **NVIDIA Groq 3 LPX** 是另一代系统，不能沿用早期芯片的容量。它的 LP30 芯片，每颗有 500 MB SRAM，一机柜 256 颗，总计 128 GB。单颗的片上 SRAM 带宽是 150 TB/s，芯片间互连标称双向聚合带宽是 2.5 TB/s。[^lpx]

这两个带宽不能不看口径就直接比，但量级差别很直观：留在芯片里读，和跨芯片传，仍然是两件事。

模型拆到很多颗芯片上，除了按层排流水线，还可能把某一层的矩阵分片并行计算，再交换中间结果。只要下一步依赖这些结果，就得等数据到齐。参数不再从 HBM 来回搬，不代表通信就消失了。

那要怎么让几百颗芯片配合好？

## 干脆在编译时，把时间表排出来

GPU 不是毫无计划地运行：kernel、数据布局和算子调度都可以事先优化。但它也依赖运行时硬件调度、缓存和访存机制，不同执行单元会遇到不同的等待，通信也可能有争用。

Groq 更进一步，把大量工作交给编译器：**哪条指令什么时候执行，数据什么时候到某个单元，哪段时间用哪条链路，尽量事先排好。** 这通常叫静态调度、确定性执行，而不是等运行起来再由硬件处理所有细节。[^tsp]

它的 SRAM 也主要作为软件显式管理的工作存储，而不是让硬件缓存自己猜“下次会用什么”。新 LPX 仍沿用这个思路，并通过芯片间时钟漂移补偿、同步机制来维持协调。[^lpx]

<figure class="inference-figure">
{% include inference/static-schedule.svg %}
<figcaption>用三颗芯片举例：编译器同时规划计算与通信，让数据在下一步需要前到达。时间格只说明调度思路，不对应真实时钟数。</figcaption>
</figure>

这样做的价值，不是“所有芯片永远不用等”，而是把很多等待变成**已知、可以安排和重叠的时间**，减少不可预测的停顿。

算完必须传的结果，还是要传；有数据依赖的地方，还是得等。只是编译器有机会把这段等待藏在别的工作后面。几百颗像一颗那样跑，是协同效果的比喻，不是说远端 SRAM 真的和本地一样快。

代价也很清楚：编译器更复杂，模型和执行形状要能映射到这种时间表。把缓存和调度做得更通用，还是把执行安排得更确定，本来就是两种设计取舍。

## NVIDIA 为什么要它？

2025 年 12 月 24 日，Groq 宣布与 NVIDIA 达成**非独占推理技术授权**；创始人 Jonathan Ross、总裁 Sunny Madra 等人加入 NVIDIA。Groq 继续独立运营，GroqCloud 也没有因此停掉。这不能简单写成“NVIDIA 买下了 Groq”。[^deal]

2026 年 8 月 24 日，NVIDIA 宣布 Groq 3 LPX 已进入全面生产。它面向的是一个很具体的需求：不光一座机房一秒能产多少 token，还要让**每个人拿到 token 的速度够快，延迟够稳**。写代码、语音交互和多步 Agent，都会在意这件事。[^launch]

至于快多少，需要分开看。GroqCloud 的 gpt-oss-120b 文档目前列的是约 500 tokens/s，这是服务规格，不是说某颗老芯片单独就能做到，也不能不加条件地拿来除以某套 H100 的数字。[^cloud]

LPX 的一个新例子更明确：NVIDIA 在 2026 年 8 月公布，Artificial Analysis 用 **Gemma 4 31B、100K 输入上下文**测得中位输出速度 **3,431 tokens/s**；当时同图最快的公开服务是 870，约为四倍。数据快照是 8 月 20 日，公告是 8 月 24 日。[^bench]

这个成绩说的是特定模型、上下文和测试口径下的输出速度，不是“所有模型都快四倍”，也不是一机柜总吞吐量。上市节奏同样要分清：进入生产，不等于所有云用户当天都能调用；公开公告点名了 Nebius 等采用者，具体可用性要看服务商。

## 更有意思的，是它怎么和 GPU 分工

SRAM 快，但容量还是紧。LPX 一柜 128 GB，里面除了权重，还要安排激活、KV 状态和运行时工作区。不能简单算“减掉 60 GB 权重，剩下都是 KV Cache”，更不能把这个结果套回早期 Groq 的配置。

GPU 刚好有另一种长处：HBM 容量更大，生态也更通用。于是，NVIDIA 的混合推理方案里，有一种很自然的分工：

<figure class="inference-figure">
{% include inference/hybrid.svg %}
<figcaption>Attention–FFN 分离的一种配置：GPU 保留 KV Cache 并计算 attention，LPX 计算 FFN / MoE，往返传的是中间激活。不是每次把整段 KV Cache 搬过去。</figcaption>
</figure>

注意力需要读取长对话的 KV Cache，就放在 GPU 侧；FFN（前馈网络）或 MoE 专家层的权重，则常驻 LPU 的 SRAM。每层交换中间激活，让两边各做擅长的部分。

这和上一篇的 prefill–decode 分离还不一样：**前者按阶段切，Attention–FFN 分离是在一轮 decode 内按计算模块切。** NVIDIA 也公开了 prefill–decode 分离、外部草稿模型推测解码等其他组合，不能把其中一种写成唯一架构。[^bench]

绕了一圈，问题又回到容量：把权重读取加速以后，KV Cache 的容量和读取成本就更显眼了。它到底存的是什么，为什么越聊越大，为什么又能让 API 便宜那么多？下一篇就讲这个。

## 资料与版本

早期 TSP、GroqCloud 服务与 NVIDIA LP30 / LPX 分别标注，避免混用代际。性能与产品状态核对于 2026-10-29。

[^tsp]: Groq，[ISCA 2022 论文](https://doi.org/10.1145/3470496.3527405)，分布式 SRAM、软件控制通信与确定性网络；220 MiB 是论文中的早期规格。
[^lpx]: NVIDIA，[Inside NVIDIA Groq 3 LPX](https://developer.nvidia.com/blog/inside-nvidia-groq-3-lpx-the-low-latency-inference-accelerator-for-the-nvidia-vera-rubin-platform/)，LP30 容量、带宽和编译器控制执行。
[^deal]: Groq，[非独占技术授权公告，2025-12-24](https://groq.com/newsroom/groq-and-nvidia-enter-non-exclusive-inference-technology-licensing-agreement-to-accelerate-ai-inference-at-global-scale)。
[^launch]: NVIDIA，[Groq 3 LPX 进入生产，2026-08-24](https://investor.nvidia.com/news/press-release-details/2026/NVIDIA-Groq-3-LPX-Now-in-Full-Production-With-World-Class-Speed-for-Agentic-AI/default.aspx)。
[^cloud]: Groq，[支持的模型及服务速度](https://console.groq.com/docs/models)。
[^bench]: NVIDIA，[Groq 3 LPX 的长上下文测试与混合推理配置](https://developer.nvidia.com/blog/how-nvidia-groq-3-lpx-unlocks-ultrafast-interactivity-at-long-context-on-nvidia-vera-rubin/)，包含测试时间、统一 tokenizer 及三种组合方式。
