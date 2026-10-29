---
title: "理解 QKV：从机器翻译到自回归 LLM"
date: 2026-10-28
permalink: /blogs/where-qkv-comes-from/
lang: zh-CN
read_time: true
read_time_minutes: 8
extra_css: /assets/css/qkv.css
excerpt: "翻译能看完整原文，为什么生成仍要遮住未来？从这两个容易混淆的问题出发，理解 Attention 的读取范围，以及 Q、K、V 的来由。"
tags:
  - llm
  - attention
  - transformer
  - deep-learning
---

<p class="qkv-intro">理解 Attention，先分清两件事：怎样读取信息，以及哪些信息允许被读取。QKV 描述读取操作，mask 约束可见范围。</p>

读过 Transformer 论文，仍可能卡在两个看似矛盾的地方：机器翻译需要完整上下文，为什么还要“只看前面”？现代 LLM 又为什么一个 token 一个 token 地生成？把原文、译文和生成过程分开，QKV 的角色也会更清楚。

## 1．翻译能看完整原文，受限的是未来译文

以标准的整句翻译为例：

> He sat on the bank of the river. → 他坐在河岸上。

开始翻译时，**完整原文已经给定，译文却还没有生成。** 翻译 bank 时，当然可以利用它后面的 river 判断词义。

原始 Transformer 使用 encoder–decoder 架构：encoder 编码完整源句；decoder 一边读取这些表示，一边生成译文。因此，三种 attention 的范围不同：[6]

<div class="qkv-article-table" role="region" aria-label="翻译模型中三种 attention 的读取范围" tabindex="0" markdown="1">

| Attention 层 | 查询来自 | 可以读取 |
|---|---|---|
| Encoder self-attention | 原文 | 完整原文，前后都能看 |
| Decoder self-attention | 译文 | 已知的译文前缀 |
| Cross-attention | 译文 | 完整原文的 encoder 表示 |

</div>

“只看前面”限制的是 **decoder 对译文的读取**，并没有把原文后半句遮住。逐词翻译可以写成：

$$
p(y\mid x)=\prod_t p(y_t\mid y_{<t},x).
$$

$x$ 是完整原文，$y_{<t}$ 是已经生成的译文。两者并不矛盾。实时同传则另当别论：原文尚未说完，模型自然只能使用已经收到的部分。

<figure class="qkv-figure">
{% include blog/qkv-context.svg %}
<figcaption><span class="qkv-caption-label">图 1</span> 翻译模型同时依赖完整原文和已知译文；典型自回归 LLM 则把提示词与已生成回答放在同一个序列前缀里。图中省略内部层结构。</figcaption>
</figure>

## 2．现代 LLM 的“只能看前面”，来自自回归建模

GPT 这类常见的 decoder-only LLM，把提示词和回答组织成一个持续延长的 token 序列，以前缀预测下一个 token：[7]

$$
p(z_1,\ldots,z_T)=\prod_t p(z_t\mid z_{<t}).
$$

所以，更准确的说法是：**先选择自回归建模，普通生成过程才需要逐 token 推进。** 不是因为界面逐字显示，模型才被迫采用 causal attention。

提示词已经完整给定，生成回答的当前位置可以读取全部提示词；不过，标准 causal LLM 在构造提示词各位置的表示时，仍然遵守从左到右的约束。

训练时整段文本都在，为什么还要遮住后文？为了不让模型偷看答案。例如输入“我 喜欢 猫”，在“喜欢”这个位置预测“猫”时，不能直接读取后面的“猫”。

<div class="qkv-article-table" role="region" aria-label="自回归模型训练与生成的区别" tabindex="0" markdown="1">

| 阶段 | 怎样计算 | 为什么需要约束 |
|---|---|---|
| 训练 | 多个位置的预测可以并行计算 | 用 causal mask 避免答案泄漏 |
| 生成 | 预测一个 token，再追加到前缀 | 后续 token 尚未确定 |

</div>

这里应称 **causal attention（因果注意力）** 或 **自回归推理**。**Causal inference（因果推断）** 是研究干预与因果效应的另一类问题。并非所有语言模型、翻译模型都必须采用自回归生成。

<h2 id="bottleneck">3．回到起点：Attention 为什么出现？</h2>

知道“能看什么”以后，再问“怎样读取”。早期一些 encoder–decoder 翻译模型把整句压成固定大小的编码，交给 decoder。[1] 这像收起原文，只靠一张笔记翻译；句子越长，保留细节越困难。

Bahdanau 等人在 2014 年发布、发表于 ICLR 2015 的工作，让 decoder 按当前需要读取各个源位置的表示 $h_j$，缓解这个瓶颈。[2]

<h3 id="soft-alignment">先打分，再按权重汇总</h3>

设 decoder 当前状态为 $s$，一次读取可以写成：

$$
\begin{aligned}
e_j&=\operatorname{score}(s,h_j),\\
\alpha_j&=\operatorname{softmax}(e)_j,\\
c&=\sum_j\alpha_jh_j.
\end{aligned}
$$

生成 cat 时多读“小猫”，生成 sits 时多读“坐在”。每一步得到不同的上下文 $c$，而不是始终依赖同一份摘要。

这叫“软”读取，因为可以混合多个位置，不必硬选一个。整个过程可微，读取方式能随翻译任务一起训练。早期打分可以用小神经网络；Luong 等人在 2015 年又研究了点积等形式。[3] **Attention 的核心是按需加权读取，点积只是其中一种实现。**

<h2 id="key-value">4．Q、K、V：为什么把一个表示分成三份？</h2>

早期计算里，三个角色已经存在：$s$ 发起查询，是 Query；$h_j$ 参与匹配，是 Key；同一个 $h_j$ 又被加权汇总，充当 Value。

**匹配依据与读取内容，为什么一定要相同？** 查询“财务部的张三”，定位需要姓名和部门，想取回的却可能是电话号码。神经网络也可以分别学习这两种表示。2015—2016 年的 Memory Networks 相关工作已体现这种分工。[4][5]

<figure class="qkv-figure">
{% include blog/qkv-read.svg %}
<figcaption><span class="qkv-caption-label">图 2</span> Q 与 K 决定权重，权重作用于对应的 V。图中的数字仅作示意。</figcaption>
</figure>

<h3 id="projections">独立投影，让“相关”不必等于“相似”</h3>

在“小猫吃了鱼”中，“吃”需要读取“小猫”的信息，但动作和实体未必应该在原始向量空间里很相似。因此，Transformer 为不同角色学习不同投影：

$$
\begin{aligned}
q_i&=x_iW_Q,\\
k_i&=x_iW_K,\\
v_i&=x_iW_V.
\end{aligned}
$$

$W_Q$ 构造查询，$W_K$ 构造供匹配的表示，$W_V$ 构造待传递的内容。Q 与 K 分开，也允许“动作寻找施事者”和反方向采用不同打分。

直接令 Q、K、V 相同也能定义 attention；独立投影是增加灵活性的设计，不是数学必然。矩阵通过任务损失共同训练，没有人预先规定某一维必须代表“主语”。

## 5．一个能手算的例子

设 $q=[1,0]$，有三个候选：

<div class="qkv-article-table" role="region" aria-label="手算示例的 Key 和 Value" tabindex="0" markdown="1">

| 位置 | Key | Value |
|---|---|---|
| A | $[2,0]$ | $[1,0]$ |
| B | $[0,2]$ | $[0,1]$ |
| C | $[1,1]$ | $[1,1]$ |

</div>

点积分数为 $[2,0,1]$。除以 $\sqrt{2}$ 后做 softmax，得到权重约 $[0.576,0.140,0.284]$。再汇总 Value：

$$
\begin{aligned}
o&=0.576v_A+0.140v_B+0.284v_C\\
 &\approx[0.860,0.424].
\end{aligned}
$$

固定 Q、K，只改 V：权重不变，读出的内容改变。固定 V，只改 Q 或 K：内容不变，混合比例改变。这就是匹配与内容传递的分工。

<h2 id="self-attention">6．Self、cross、causal：三个不同的问题</h2>

如果 Q 来自译文、K 和 V 来自原文，就是 cross-attention。如果三者来自同一序列的表示，就是 self-attention：

<figure class="qkv-figure">
{% include blog/qkv-self.svg %}
<figcaption><span class="qkv-caption-label">图 3</span> 同一份 X 分别生成 Q、K、V。“Self”说明信息来源相同，不意味着只读自己或只能看前文。图中省略 mask。</figcaption>
</figure>

把所有查询一起计算，加上可见范围的 mask $M$，就是：

<div class="qkv-formula">
$$
O=\operatorname{softmax}_{\mathrm{row}}
\left(\frac{QK^\top}{\sqrt{d_k}}+M\right)V.
$$
</div>

$QK^\top$ 负责打分；$M$ 在允许位置取 0，禁止位置取 $-\infty$，使对应 softmax 权重变成 0；乘以 V 完成读取。缩放控制分数尺度，多头则让多组投影分别读取，再组合结果。[6]

因此，理解一个 attention 层，可以依次问：**谁在查、从哪里读、允许读哪些位置？** QKV 定义读取角色，self/cross 区分来源，causal mask 规定可见范围。机器翻译能看完整原文，与 decoder 逐词生成译文，恰好是在回答不同的问题。

## 参考文献

<div class="qkv-references" markdown="1">

1. Sutskever et al. [Sequence to Sequence Learning with Neural Networks](https://arxiv.org/abs/1409.3215). 2014.
2. Bahdanau et al. [Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473). 2014 / ICLR 2015.
3. Luong et al. [Effective Approaches to Attention-based Neural Machine Translation](https://aclanthology.org/D15-1166/). 2015.
4. Sukhbaatar et al. [End-To-End Memory Networks](https://arxiv.org/abs/1503.08895). 2015.
5. Miller et al. [Key-Value Memory Networks for Directly Reading Documents](https://arxiv.org/abs/1606.03126). 2016.
6. Vaswani et al. [Attention Is All You Need](https://arxiv.org/abs/1706.03762). 2017.
7. Hugging Face. [Causal language modeling](https://huggingface.co/docs/transformers/en/tasks/language_modeling).

</div>
