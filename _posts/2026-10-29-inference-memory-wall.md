---
title: "GPU 和 HBM"
date: 2026-10-29 09:00:00 -0700
permalink: /blogs/inference-memory-wall/
lang: zh-CN
series: inference
series_order: 2
read_time: true
read_time_minutes: 8
excerpt: "拿 H100 和一个 8B 模型算一笔账：计算只要 0.016 毫秒，搬参数却要 4.8 毫秒。prefill、decode 和 batching 的区别，就在这里。"
tags: [llm, inference, gpu]
---

我以前看 GPU，最关心的是算力：有多少 Tensor Core，一秒能做多少次运算。后来才发现，大模型推理有时候根本不是算不动，而是数据还没送到，计算单元已经在等了。

拿一块 NVIDIA H100 SXM 80GB 来举例，参数主要放在计算芯片旁边的显存里，计算在芯片里面做。算到某一层时，要把这一层用到的参数分块送进来。问题就在于：**算得快，不代表喂得饱。**

## 显存与 HBM

首先区分一下显存与 HBM。严格地说，High Bandwidth Memory (HBM) 是显存的一种实现，它本质上仍然是 DRAM。比如，H100 SXM 用的是 HBM3，但不能说显存就是 HBM。本文为了方便，就暂时混用这两个概念了。

为什么 GPU 要用 HBM？容量大是一方面，更关键的是需要**在有限功耗和封装面积里，把数据搬得足够快**。HBM 把多层 DRAM 堆起来，通过很宽的接口连接计算芯片。它不是普通内存的反义词，而是一种优先追求高带宽的内存设计。三星、SK 海力士和美光是目前全球最主要的三家供应商。[^hbm]

<figure class="inference-figure">
{% include inference/gpu-memory.svg %}
<figcaption>参数从 HBM 分块读进计算芯片。芯片里也有缓存和工作存储，但容不下整个大模型。</figcaption>
</figure>

### 区分“片上／片外”

真正参与运算的数据，当然得先到达计算电路。这里说的“片上／片外”，特指相对于**计算裸片（compute die）**的位置，不是相对于整张显卡。HBM 本身也是芯片，可以和 GPU 计算裸片放在同一个封装里，但仍是不同的硅片。

区别在于：**临时搬进来算，还是一直留在里面。** 比如一个 16 GB 的模型，GPU 逐层、分块读取权重；算到后面时，前面的大部分权重已经没有空间留在片内。下一轮生成又从前面的层开始，于是还得从 HBM 重新读。上一轮“进过芯片”，不代表下一轮需要时还在。

后面讲 Cerebras、Groq 时，“参数放在片上 SRAM”指的是为权重安排足够的片上存储，让它们跨生成步骤持续驻留。整套模型可以分布在多颗芯片的 SRAM 里，不要求一颗装完。计算时仍要从 SRAM 读到计算单元，**省下的是反复跨片外接口的搬运，不是从此没有数据移动。**

## 以 8B 模型距离

H100 SXM 启用了 132 个 SM（Streaming Multiprocessor，流式多处理器），每个 SM 里有 4 个 Tensor Core，专门加速矩阵乘加。当然，SM 里还有普通运算单元、寄存器、共享内存等，不能把它只看成四个矩阵计算器。[^hopper]

假设我们跑的是一个 **80 亿参数的 Transformer 模型**，权重使用 FP16 或 BF16，也就是每个参数 **16 bit = 2 字节**。假设只考虑权重，暂不考虑 KV Cache、激活和运行时开销：

$$
8\times10^9\times2\text{ bytes}=16\text{ GB}.
$$

H100 SXM 的标称 HBM 带宽是 3.35 TB/s。理想情况下，把 16 GB 权重读一遍，需要：

$$
\frac{16\text{ GB}}{3350\text{ GB/s}}\approx4.8\text{ ms}.
$$

再看计算。单个 token 的主要权重计算量可以粗估为 $2P$ FLOPs：一次乘法加一次加法，算两次浮点运算。这里 $P$ 是参数量。

H100 SXM 的 **FP16/BF16 Tensor Core 峰值**约为 989 TFLOPS。于是，只按峰值计算：

$$
\frac{2\times8\times10^9}{989\times10^{12}}
\approx0.016\text{ ms}.
$$

**搬参数约 4.8 ms，做这些乘加的峰值时间约 0.016 ms，差了接近 300 倍。**

## 很自然的想法：那就把同一份参数多用几次

假如参数搬进来以后，不是只给一个 token 用，而是同时给 300 个 token 用，搬运成本就可以摊开。这对应的术语叫**算术强度**：每搬一个字节，做多少计算。

对于刚才的 16-bit 权重，若一次矩阵计算处理 $B$ 个 token，忽略其他流量，算术强度大约就是 $B$ FLOPs/byte。而 H100 的峰值算力除以带宽，大约是 295 FLOPs/byte。于是才大约有了“要复用 300 次，才有机会从等内存变成忙计算”这个目标。

训练时，一次通常会处理很多 token，同一份权重能反复使用，所以大块矩阵乘法更有机会吃满算力。当然，训练也可能卡在通信、激活读写等地方，不能一概而论。

推理则要分两步看：prefill 和 decode。

<figure class="inference-figure">
{% include inference/prefill-decode.svg %}
<figcaption>prefill 同时处理已知输入；普通自回归 decode 要等上一轮的输出，才能确定下一轮的输入。不同用户之间没有这条依赖，可以拼到一起算。</figcaption>
</figure>

### Prefill：输入都知道，可以一起算

你贴进去的一大段文章，token 已经全部给定。虽然模型有 causal masking，每一层仍然可以并行处理这些位置，不必等“读完第一个词”再启动第二个词。

因此，几百上千个输入 token 可以一起用这份权重，矩阵计算更容易发挥出来。这也解释了为什么贴一大段文字进去，模型往往很快就能处理完。

### Decode：一个人用，每一步只有一个新位置

decode 就不一样了。普通自回归生成中，下一个 token 是什么，要等上一个生成出来才知道。因此，**对单个请求，一次 decode 通常只推进一个新 token**。

在小 batch、上下文还不特别长的情况下，每轮都要读取大量权重，却只处理很少的新 token。离刚才“复用几百次”的目标差得很远。

那参数能不能留在芯片里，别再搬了？

芯片里确实有存储。H100 SXM 的 L2 缓存是 50 MB，再加上分布在各 SM 的寄存器、共享内存等，远不足以装下 LLM。所以，模型每生成一个 token，大部分权重通常还得重新经过 HBM 接口；缓存和融合可以减少局部流量，但不能凭空留住整套参数。[^hopper]

## Batching

服务商会把很多人的请求放到一起算，叫 batching。每个人仍然只生成自己的下一个 token，但这一步，几十上百个人可以共享一次权重读取。实际服务还常用 continuous batching：有人结束就退出，有新请求就补进来。[^phases]

这样一块卡能服务更多人，整体吞吐量更高。

读到这里，下一步的问题就很自然了：既然往返 HBM 这么费事，能不能干脆把模型一直放在计算芯片里面？下面两篇文章，我们就探索这个问题。

## 资料与计算口径

本文的算例使用十进制 GB/TB、16-bit 稠密权重和稠密 Tensor Core 峰值，仅估算主要权重乘加与理想权重读取；不适用于直接估算 MoE 的全部权重流量。规格核对于 2026-10-29。

[^hbm]: [Micron：HBM 的堆叠、接口与带宽](https://www.micron.com/products/memory/hbm)；[TrendForce：三家主要 HBM 供应商](https://www.trendforce.com/presscenter/news/20260213-12929.html)。
[^hopper]: NVIDIA，[Hopper Architecture In-Depth](https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/)，H100 SXM 的 SM、Tensor Core、L2 和片上存储规格。
[^spec]: NVIDIA，[H100 产品规格](https://www.nvidia.com/en-eu/data-center/h100/)。规格表的星号注明 with sparsity，本文取约一半作为稠密 FP16/BF16 峰值。
[^roofline]: NVIDIA，[AI Model Co-Design: Hardware-Friendly LLM Design](https://developer.nvidia.com/blog/ai-model-co-design-hardware-friendly-llm-design/)，算术强度与 compute-bound / memory-bound 的关系。文中 295 和时间比由上述规格自行计算。
[^phases]: NVIDIA，[Mastering LLM Techniques: Inference Optimization](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)，prefill、decode 与 continuous batching。
