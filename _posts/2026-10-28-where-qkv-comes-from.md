---
title: "Q、K、V 是怎么来的"
date: 2026-10-28
permalink: /blogs/where-qkv-comes-from/
lang: zh-CN
series: inference
series_order: 1
read_time: true
read_time_minutes: 8
extra_css: /assets/css/qkv.css
excerpt: "从机器翻译中固定长度 representation 的瓶颈，到按需读取、Key–Value 分工与 self-attention：沿着问题理解 QKV，而不是从公式开始背诵。"
tags:
  - llm
  - attention
  - transformer
  - deep-learning
---

<p class="qkv-intro">为什么每个 token 已经有一个向量，Attention 还要把它变成 Query、Key、Value 三份？</p>

一言以概之，QKV 不是数学定理要求的唯一结构，而是围绕一个问题逐渐形成的设计：**怎样让神经网络根据当前需要，从一组信息里读出有用的内容？** 沿着机器翻译这条线，对 QKV 的理解就清晰多了。

<h2 id="bottleneck">摒弃固定长度 representation</h2>

早期一种典型的机器翻译方案是 encoder–decoder：encoder 用 RNN 或 LSTM 读完源句，把它压成固定维度的 representation $c$；decoder 以此为条件，逐词生成译文。2014 年的 Seq2Seq 是其中的重要代表。[\[1\]](#qkv-ref-1)

注意与 ChatGPT 区分，这里逐词生成的是**译文**，不是逐词获得原文。翻译开始时，完整源句已经给定，翻译过程可以 access 所有源句的词。问题的关键是，decoder 如何去利用这些信息。

在老派的机器翻译，比如 Seq2Seq，它们的工作方式像是读完原文后把原文收起来，只带着一张固定大小的笔记完成翻译。句子越长，细节自然越难保留。那么，一个自然的改进就是：**把原文每个位置的表示都留下，让 decoder 需要时回头读取。**

<figure class="qkv-figure">
{% include blog/qkv-bottleneck.svg %}
<figcaption><span class="qkv-caption-label">图 1</span> 从共用一个句子 representation，改成每一步按需读取。h₁、h₂、h₃ 是包含上下文的源位置表示，cₜ 是当前步骤读出的信息。</figcaption>
</figure>

<h2 id="soft-alignment">Attention Is All You Need</h2>

把“小猫坐在垫子上”翻译成“The cat sits on the mat”，生成 cat 时应该更多参考“小猫”，生成 sits 时则更多参考“坐在”。我们希望模型学会类似这样的权重：

<div class="qkv-article-table" role="region" aria-label="示意翻译对齐权重" tabindex="0" markdown="1">

| 当前生成 | 小猫 | 坐在 | 垫子上 |
|---|---:|---:|---:|
| cat | **0.85** | 0.10 | 0.05 |
| sits | 0.10 | **0.80** | 0.10 |
| mat | 0.05 | 0.10 | **0.85** |

</div>

实现时，用 decoder 当前状态 $s$ 与每个源位置的表示 $h_j$ 打分，再用 softmax 转为权重：

$$
e_j=\operatorname{score}(s,h_j),\qquad
\alpha_j=\frac{\exp(e_j)}{\sum_r\exp(e_r)}.
$$

然后读取一份适合当前步骤的上下文：

$$
c=\sum_j\alpha_jh_j.
$$

decoder 用它来帮助预测目标词。下一步状态变了，读取重点也随之改变。Bahdanau 等人在 2014 年发布、发表于 ICLR 2015 的工作，让模型把翻译与这种“软对齐”一起学出来。[\[2\]](#qkv-ref-2)

“软”是指不必只选一个位置，而是连续地混合读取多个位置。整条计算链可以 differentiate，翻译损失可以 BP，通常不需要人工标注每个词该对齐哪里。

最初的打分可以由一个小神经网络完成；2015 年 Luong 等人又研究了点积等形式。[\[3\]](#qkv-ref-3) **Attention 的核心是按需加权读取，点积只是打分的一种实现。**

<h2 id="key-value">Q、K、V 是怎么来的</h2>

在刚才的过程中，其实三个角色已经出现了：

<div class="qkv-article-table" role="region" aria-label="早期 attention 与 QKV 的对应" tabindex="0" markdown="1">

| 角色 | 在早期翻译 attention 中 |
|---|---|
| Query：发起读取 | Decoder 当前状态 $s$ |
| Key：供查询匹配 | 源位置表示 $h_j$ |
| Value：被加权读取 | 还是同一个 $h_j$ |

</div>

**早期的 $h_j$ 同时承担了 Key 和 Value：既决定“读哪里”，又充当“读出的内容”。** 但这两件事不一定要用同一套表示。研究发现，神经网络也可以分别学习适合匹配和适合传递的表示。2015 年的 End-to-End Memory Networks、2016 年的 Key-Value Memory Networks 已体现了这种分工。[\[4\]](#qkv-ref-4)[\[5\]](#qkv-ref-5)

<figure class="qkv-figure">
{% include blog/qkv-read.svg %}
<figcaption><span class="qkv-caption-label">图 2</span> Q 与 K 决定权重，再用这些权重汇总对应的 V。权重为人为构造的示意值。</figcaption>
</figure>

于是，QKV 可以理解为一次“软查询”：**Q 提供需求，K 用于定位，V 提供内容。** 它不像普通字典只返回精确匹配的一项，而是对多个候选做加权汇总。

假设位置 $i$ 的表示是行向量 $x_i$，直接令 $q_i=k_i=v_i=x_i$，当然能做 attention。但是，独立投影的意义在于给不同角色更多自由：

$$
\begin{aligned}
q_i&=x_iW_Q,\\
k_i&=x_iW_K,\\
v_i&=x_iW_V.
\end{aligned}
$$

<h2 id="self-attention">Self-attention：让同一句话内部相互读取</h2>

早期翻译 attention 的查询来自译文 decoder，被读取的信息来自源句 encoder。再往前一步：同一句话的每个位置，也可以查询其他位置。

例如“小猫吃完鱼后，它睡着了”，各个位置可以通过读取其他位置，形成更有用的表示。Self-attention 本身并没有规定只能看前文。令上一层各位置的表示组成矩阵 $X$，同一份输入分别产生 Q、K、V：

<figure class="qkv-figure">
{% include blog/qkv-self.svg %}
<figcaption><span class="qkv-caption-label">图 3</span> “Self”指查询与被读取的信息来自同一组位置，不是只关注自己。图中画的是一个 head。</figcaption>
</figure>

把所有位置的查询一起计算，就得到熟悉的公式：

<div class="qkv-formula">
$$
O=\operatorname{softmax}_{\mathrm{row}}
\left(\frac{QK^\top}{\sqrt{d_k}}\right)V.
$$
</div>

$QK^\top$ 是位置之间的打分表：第 $i$ 行表示查询 $i$ 对各候选位置的分数。逐行 softmax 后，乘以 V 完成读取。除以 $\sqrt{d_k}$ 用于控制点积尺度，避免 softmax 过于饱和；多头则让多组投影分别读取，再组合结果。[\[6\]](#qkv-ref-6)

### 机器翻译 vs. GPT

典型的 encoder–decoder 用于机器翻译时，可以看完整源句；只有正在生成的译文存在限制。

与之相对，GPT 这类常见的自回归 LLM 则采用 decoder-only 架构，把提示词和已生成回答放进同一个序列前缀，预测下一个 token。[\[7\]](#qkv-ref-7)它们先选择自回归建模，普通推理生成时才需要逐 token 推进。也就是说，**回答的当前位置能读取完整提示词，却不能读取尚未生成的后续回答。**训练时虽然整段文本都在，causal mask 仍要防止模型偷看答案：例如在“喜欢”这个位置预测“猫”，就不能直接读取后面的“猫”。因此，训练可以并行计算多个位置的预测，普通生成则根据已知前缀依次推进。

回到机器翻译，原始 Transformer 的三个 attention 层，读取范围并不相同：[\[6\]](#qkv-ref-6)

<div class="qkv-article-table" role="region" aria-label="机器翻译中三种 attention 的读取范围" tabindex="0" markdown="1">

| Attention 层 | Q 来自哪里 | K、V 的可见范围 |
|---|---|---|
| Encoder self-attention | 源句 | 完整源句，前后都能看 |
| Decoder self-attention | 译文 | 已知的译文前缀 |
| Encoder–decoder attention（cross-attention） | 译文 | 完整源句 |

</div>

例如翻译 bank 时，完全可以利用源句后面的 river 判断它是“河岸”。但如果逐词生成译文，第 $t$ 个词还没生成时，后面的译文自然不可用。这个自回归模型写成：

$$
p(y\mid x)=\prod_t p(y_t\mid y_{<t},x).
$$

$x$ 是完整源句，$y_{<t}$ 才是受限的译文前缀。训练时虽然有完整参考译文，也要把 decoder 输入右移，并用 causal mask 遮住后续位置，避免模型偷看待预测的答案；实现上是在 softmax 前把禁止读取的位置设为 $-\infty$。[\[6\]](#qkv-ref-6)

这里的 **causal attention** 指这种单向读取约束，与统计学里研究干预、因果效应的 **causal inference（因果推断）** 不同。逐词生成也是模型的选择，并非机器翻译任务本身要求所有模型都只能这样做。

在 2017 年，Transformer 横空出世，把 self-attention 放到了 sequential processing 的中心，结合 positional encoding、前馈网络等组件构成完整架构。[\[6\]](#qkv-ref-6) 它不是 attention 的起点。

## 参考文献

<div class="qkv-references" markdown="1">

1. <span id="qkv-ref-1"></span>Sutskever et al. [Sequence to Sequence Learning with Neural Networks](https://arxiv.org/abs/1409.3215). 2014.
2. <span id="qkv-ref-2"></span>Bahdanau et al. [Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473). 2014 / ICLR 2015.
3. <span id="qkv-ref-3"></span>Luong et al. [Effective Approaches to Attention-based Neural Machine Translation](https://aclanthology.org/D15-1166/). 2015.
4. <span id="qkv-ref-4"></span>Sukhbaatar et al. [End-To-End Memory Networks](https://arxiv.org/abs/1503.08895). 2015.
5. <span id="qkv-ref-5"></span>Miller et al. [Key-Value Memory Networks for Directly Reading Documents](https://arxiv.org/abs/1606.03126). 2016.
6. <span id="qkv-ref-6"></span>Vaswani et al. [Attention Is All You Need](https://arxiv.org/abs/1706.03762). 2017.
7. <span id="qkv-ref-7"></span>Hugging Face. [Causal language modeling](https://huggingface.co/docs/transformers/en/tasks/language_modeling).

</div>
