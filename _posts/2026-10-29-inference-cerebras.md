---
title: "Cerebras"
date: 2026-10-29 09:01:00 -0700
permalink: /blogs/inference-cerebras/
lang: zh-CN
series: inference
series_order: 3
read_time: true
read_time_minutes: 4
excerpt: "既然搬参数这么慢，为什么不直接把它们留在芯片里？Cerebras 把芯片做到晶圆那么大，但容量、通信和 KV Cache 的账还得继续算。"
tags: [llm, inference, cerebras]
---

如果不用 HBM，一开始就把参数全部放在计算芯片里呢？Groq 和 Cerebras 的就是这样做 inference 的。这篇先讲 Cerebras。

Cerebras 快得很夸张。它发布 gpt-oss-120b 服务时，曾公布 Artificial Analysis 测得约 2,700 output tokens/s。8张 Nvidia H100 一起跑，才 250 个左右。[^speed]

## SRAM

SRAM 的全名是 Static Random-Access Memory。“Static”不是说内容不会变，而是它不需要像 DRAM 那样周期性刷新。代价是每个 bit 占的硅面积大得多。

GPU 也有 SRAM，只是主要拿来做寄存器、缓存和共享内存。Cerebras 的推理系统则让模型权重常驻在分布式片上 SRAM 中，就在计算核心附近，省掉每生成一个 token 就从外部 HBM 重读权重的那段路。[^inference]

一颗普通大小的计算芯片，做出几百 MB 的 SRAM 已经很可观。几十 GB 的权重，要靠很多颗芯片的面积才装得下。如果把这些芯片切开，再通过封装、板级走线或线缆互连，跨芯片的数据传递又会成为问题。

既然如此，干脆直接做一颗大芯片！

## 大到一块晶圆的芯片

先补充一点光刻机背景，常规光刻不是把整个晶圆一次照出来，而是一块曝光场、一块曝光场地做。以 ASML 的一款 0.33 NA EUV 设备为例，最大曝光场是 **26 × 33 mm**。H100 的 GH100 计算裸片约 814 mm²，已经接近这个面积量级。[^reticle]

那把光刻机做大，不就行了？

原则上可以扩大曝光场，26 × 33 mm 不是一条永远不能突破的物理定律。但要扩大的不是机器外壳，而是**能保持足够精度的成像范围**。光刻机要把掩模图案投影到晶圆上，范围更大，还得让各处的分辨率、畸变和多层图案的对准精度都达标。

这有点像投影仪：画面放大容易，放大以后每个角落还要一样清楚，就难了。一个例子是 High-NA EUV：为了提高分辨率，ASML 使用更大的反射镜，又得解决光线照到掩模的角度问题。最后采用两个方向不同的缩小倍率，兼容原有掩模，曝光场面积反而减半。[^optics]

更大掩模的路线也有人在推进。ASML 和台积电在 2026 年 9 月宣布了相关计划，但要一起升级掩模制造、检测、搬运和曝光系统，可能得 2030 年以后才能问世。[^large-mask] 而且，就算能曝光更大，巨型芯片仍要解决良率、长距离通信、供电和散热等等一系列问题。

### 另一条思路 - 把曝光区连起来

不换成一台能一次照出整片芯片的机器，也可以通过专门设计，让多个曝光区连成一个处理器。

Cerebras 就走了这条路。用很口语的话说：**别人做好以后切开，它让很多曝光区继续连在一起。** 在原本要分开的区域之间做跨 reticle 连接，形成一个约 21.5 cm 见方的巨型处理器。不是把买来的小芯片重新粘起来，而是在制造时就把它们连成整体。[^wafer]

<figure class="inference-figure">
{% include inference/wafer.svg %}
<figcaption>左：常规路线，把晶圆上的裸片切开。右：wafer-scale 路线，通过跨曝光场连接，保留一个巨大的连续处理器。网格仅为示意。</figcaption>
</figure>

以 WSE-3 为例，面积是 46,225 mm²，约为 H100 裸片的 57 倍；启用 90 万个 AI 计算核心，片上 SRAM 合计 44 GB。[^wse]

然而，晶圆不是完美的，一定会有制造缺陷。芯片越大，碰到缺陷的概率越高。台积电之所以独步天下，关键护城河就是它的良率很高。

Cerebras 的办法是把容错做细：核心很小，加上冗余核心和可以绕路的互连，坏的部分关掉，由冗余资源补上。它公开的 WSE-3 数据是约 **97 万个物理核心，但只启用 90 万个**。坏点不会自动只影响一个核心，但设计会尽量把损失局限在小范围。[^yield]

## 44 GB 还是装不下

那就用多片晶圆，进行软件层面的优化。

比如，一个 80 层模型，分到 4 片晶圆上，每片负责连续 20 层。算完第 20 层，把 activation 传给下一片，而不是把下一片的权重搬到这一片。

<figure class="inference-figure">
{% include inference/wafer-pipeline.svg %}
<figcaption>假想的 80 层模型，按连续层分成四个阶段。一次前向沿着三个阶段边界传递激活；不包含回传生成 token、控制通信等。真实划分要服从每层的权重和缓存容量。</figcaption>
</figure>

这叫**流水线并行**。这里一次前向主要经过三个跨片边界，不等于“整次请求只通信三次”。而且单个请求按顺序走，流水线其他阶段可能空着；要让整套系统忙起来，可以让不同阶段同时处理不同请求。[^pipeline]

## 速度换来了，容量的账还在

参数不搬了，KV Cache 还要占地方。它保存每个请求已经算出的注意力状态，后面生成时会继续读。

所以同样一片 44 GB SRAM，不能全部拿来放权重，还要留给 buffer、activation 和 runtime。而 8 张 80 GB H100 的名义 HBM 容量合计是 640 GB，相较之下，要宽裕得多。

## Get the Best of Two Worlds

AWS 与 Cerebras 已宣布合作：Trainium 做 prefill，算好 KV Cache，经 EFA 网络交给 Cerebras，由它做 decode。AMD 也宣布了 Helios 与 Cerebras 的分离式推理合作。[^partners]

这叫 **prefill–decode disaggregation**，把预填充与解码部署到不同硬件上。输入并行算，交给擅长吞吐的那一边；后面一个 token 一个 token 往外吐，交给擅长低延迟的那一边。

据最新新闻，Cerebras 在 2026 年 8 月给出的计划是 AMD 方案于第四季度投入生产，AWS 分离式方案预计 2027 年第一季度进入 Bedrock。[^rollout]

## Reference

[^speed]: Cerebras，[OpenAI GPT-OSS-120B Runs Fastest on Cerebras](https://www.cerebras.ai/blog/openai-gpt-oss-120b-runs-fastest-on-cerebras)，发布时的服务测试快照，不是硬件恒定速度。
[^inference]: Cerebras，[Introducing Cerebras Inference](https://www.cerebras.ai/blog/introducing-cerebras-inference-ai-at-instant-speed)。
[^reticle]: ASML，[TWINSCAN NXE:3600D](https://www.asml.com/en/products/euv-lithography-systems/twinscan-nxe-3600d)；NVIDIA，[GH100 裸片面积](https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/)。
[^wafer]: Cerebras，[S-1 披露的跨 reticle 互连与容错设计](https://www.sec.gov/Archives/edgar/data/2021728/000162828024041596/cerebras-sx1.htm)。
[^wse]: Cerebras，[第三代 Wafer-Scale Engine 发布](https://www.cerebras.ai/press-release/cerebras-announces-third-generation-wafer-scale-engine)。
[^yield]: Cerebras，[100x Defect Tolerance: How Cerebras Solved the Yield Problem](https://www.cerebras.ai/blog/100x-defect-tolerance-how-cerebras-solved-the-yield-problem)。
[^pipeline]: Cerebras，[How We Deploy Trillion-Parameter Frontier Models](https://www.cerebras.ai/blog/how-we-deploy-trillion-parameter-frontier-models-on-cerebras)，流水线、micro-batch 与缓存容量的关系。
[^partners]: [AWS 与 Cerebras 合作公告，2026-03-13](https://press.aboutamazon.com/aws/2026/3/aws-and-cerebras-collaboration-aims-to-set-a-new-standard-for-ai-inference-speed-and-performance-in-the-cloud)；[AMD 合作公告，2026-07-23](https://ir.amd.com/news-events/press-releases/detail/1293/amd-and-cerebras-announce-industry-leading-ultra-low-latency-and-high-throughput-ai-inference-solution)。
[^rollout]: Cerebras，[2026 年第二季度业务更新，2026-08-12](https://investors.cerebras.ai/news-releases/news-release-details/cerebras-systems-fast-inference-cloud-business-nearly-quadruples)。
[^cs4]: Cerebras，[CS-4 发布，2026-08-18](https://investors.cerebras.ai/news-releases/news-release-details/cerebras-unveils-cs-4-30-times-faster-gpu-based-solutions)。

[^optics]: ASML，[5 Things You Should Know About High NA EUV](https://www.asml.com/en/company/stories/2024/5-things-high-na-euv)，成像光学系统、掩模与曝光场之间的取舍。
[^large-mask]: ASML 与台积电，[大尺寸 High-NA EUV 掩模计划，2026-09-08](https://investor.asml.com/news-releases/news-release-details/asml-and-tsmc-announce-initiative-pioneer-industry-transition-large-format-photomasks-high-na-euv)。
