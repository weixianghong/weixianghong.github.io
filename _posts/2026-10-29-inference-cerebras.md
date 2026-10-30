---
title: "Cerebras：那就把参数放进芯片里"
date: 2026-10-29 09:01:00 -0700
permalink: /blogs/inference-cerebras/
lang: zh-CN
series: inference
series_order: 3
read_time: true
read_time_minutes: 6
excerpt: "既然搬参数这么慢，为什么不直接把它们留在芯片里？Cerebras 把芯片做到晶圆那么大，但容量、通信和 KV Cache 的账还得继续算。"
tags: [llm, inference, cerebras]
---

上一篇算完 H100 的账，我的第一反应是：那如果不用 HBM，一开始就把参数全部放在计算芯片里呢？

Groq 和 Cerebras 的推理路线，都在认真做这件事。这篇先讲 Cerebras。

它确实能快得很夸张。Cerebras 发布 gpt-oss-120b 服务时，曾公布 Artificial Analysis 测得约 2,700 output tokens/s。[^speed] 但这个数字不能直接拿来证明“同样模型固定比 8 张 H100 快几倍”：精度、输入长度、并发量、软件版本都得对齐。**单个请求出字快，和整套系统总吞吐量高，也不是同一个指标。**

我们先不追排行榜，看看它为什么能快。

## SRAM 很快，就是太占地方

GPU 也有 SRAM，只是主要拿来做寄存器、缓存和共享内存。Cerebras 的推理系统则让模型权重常驻在分布式片上 SRAM 中，就在计算核心附近，省掉每生成一个 token 就从外部 HBM 重读权重的那段路。[^inference]

SRAM 的全名是 Static Random-Access Memory。“Static”不是说内容不会变，而是它不需要像 DRAM 那样周期性刷新。代价是每个 bit 占的硅面积大得多。

一颗普通大小的计算芯片，做出几百 MB 的 SRAM 已经很可观。几十 GB 的权重，要靠很多颗芯片的面积才装得下。如果把这些芯片切开，再通过封装、板级走线或线缆互连，跨芯片的数据传递又会成为问题。

那为什么不直接做一颗大芯片？

## 大到一块晶圆，行不行？

常规光刻不是把整个晶圆一次照出来，而是一块曝光场、一块曝光场地做。以 ASML 的一款 0.33 NA EUV 设备为例，最大曝光场是 **26 × 33 mm**。H100 的 GH100 计算裸片约 814 mm²，已经接近这个面积量级。[^reticle]

这不是“超过这个尺寸物理上就不可能”，而是常规单曝光场芯片的制造边界。真想跨过去，需要专门解决跨曝光场连接、供电、散热和良率。

Cerebras 就走了这条路。用很口语的话说：**别人做好以后切开，它让很多曝光区继续连在一起。** 在原本要分开的区域之间做跨 reticle 连接，形成一个约 21.5 cm 见方的巨型处理器。不是把买来的小芯片重新粘起来，而是在制造时就把它们连成整体。[^wafer]

<figure class="inference-figure">
{% include inference/wafer.svg %}
<figcaption>左：常规路线，把晶圆上的裸片切开。右：wafer-scale 路线，通过跨曝光场连接，保留一个巨大的连续处理器。网格仅为示意。</figcaption>
</figure>

以 WSE-3 为例，面积是 46,225 mm²，约为 H100 裸片的 57 倍；启用 90 万个 AI 计算核心，片上 SRAM 合计 44 GB。这里的小核心和 NVIDIA 的 SM 不是同一种单位，不能直接用核心数量比算力。[^wse]

而且，晶圆不是完美的，一定会有制造缺陷。芯片越大，碰到缺陷的概率越高。不切开，坏一小块，难道整片就报废？

Cerebras 的办法是把容错做细：核心很小，加上冗余核心和可以绕路的互连，坏的部分关掉，由冗余资源补上。它公开的 WSE-3 数据是约 **97 万个物理核心，启用 90 万个**。坏点不会自动只影响一个核心，但设计会尽量把损失局限在小范围。[^yield]

## 44 GB 还是装不下，怎么办？

那就用多片晶圆。只是这一次，尽量让**权重留在原地，跨晶圆传中间结果**。

举个只为说明原理的例子：一个 80 层模型，分到 4 片晶圆上，每片负责连续 20 层。算完第 20 层，把 activation 传给下一片，而不是把下一片的权重搬到这一片。

<figure class="inference-figure">
{% include inference/wafer-pipeline.svg %}
<figcaption>假想的 80 层模型，按连续层分成四个阶段。一次前向沿着三个阶段边界传递激活；不包含回传生成 token、控制通信等。真实划分要服从每层的权重和缓存容量。</figcaption>
</figure>

这叫**流水线并行**。这里一次前向主要经过三个跨片边界，不等于“整次请求只通信三次”。而且单个请求按顺序走，流水线其他阶段可能空着；要让整套系统忙起来，可以让不同阶段同时处理不同请求。[^pipeline]

片外互连当然比片内数据通路难做，但不能拿“所有核心的片内聚合带宽”和“一条片外链路”相除，就认定程序会慢十几万倍。关键是：要传多少数据，跨多少次边界，能不能把通信和计算重叠起来。

## 速度换来了，容量的账还在

参数不搬了，KV Cache 还要占地方。它保存每个请求已经算出的注意力状态，后面生成时会继续读。我们下一篇再展开它，先记住：**模型权重可以由很多请求共享，KV Cache 通常随各自的上下文增长。**

所以同样一片 44 GB SRAM，不能全部拿来放权重，还要留给缓存、激活和运行时。如果某种模型配置每个 token 的 KV 要 320 KiB，那么 1 万 token 就约 3.05 GiB；换成别的模型、精度或注意力结构，这个数会差很多，不能把“1 万 token = 3 GB”当定律。

8 张 80 GB H100 的名义 HBM 容量合计是 640 GB，但这也是分布式容量，怎么分配、是否复制，都取决于并行方案。单看容量，确实比几片晶圆宽裕；单看这个数字，却不能直接推出两套系统各能服务多少人。

Cerebras 自己也在做流水线和 micro-batching，不能理解成“只能一个人用”。更准确的是：**SRAM 容量会限制并发；追求单请求速度，和追求更高总吞吐，仍然要取舍。**[^pipeline]

## 所以，可以让两种芯片各干一半

AWS 与 Cerebras 已宣布合作：Trainium 做 prefill，算好 KV Cache，经 EFA 网络交给 Cerebras，由它做 decode。AMD 也宣布了 Helios 与 Cerebras 的分离式推理合作。[^partners]

这叫 **prefill–decode disaggregation**，把预填充与解码部署到不同硬件上。输入并行算，交给擅长吞吐的那一边；后面一个 token 一个 token 往外吐，交给擅长低延迟的那一边。

当然，KV Cache 的交接不是免费的。需要匹配模型、精度、缓存格式，还要有足够好的网络，不能理解成两台机器接根线就行。合作宣布也不等于所有服务已经上线：Cerebras 在 2026 年 8 月给出的计划是 AMD 方案于第四季度投入生产，AWS 分离式方案预计 2027 年第一季度进入 Bedrock。[^rollout]

到这里我觉得最有意思的是：GPU 和这类推理芯片，不一定是谁取代谁。把两步分开，可能比让同一种硬件硬扛到底更合适。

## 资料与版本

这里主要用 WSE-3 解释原理。2026 年 8 月发布的 CS-4 使用 WSE-3T，仍是 44 GB SRAM，但带宽与计算规格有升级，别把两代数据混着用。[^cs4] 产品信息核对于 2026-10-29；示例分层和缓存数字为原理演示。

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
