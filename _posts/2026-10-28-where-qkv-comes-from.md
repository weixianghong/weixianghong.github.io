---
title: "Q、K、V 从何而来：Attention 的演化与直觉"
date: 2026-10-28
permalink: /blogs/where-qkv-comes-from/
lang: zh-CN
read_time: true
read_time_minutes: 16
extra_css: /assets/css/qkv.css
excerpt: "从机器翻译的固定编码瓶颈，到按需读取、Key–Value 分工与 self-attention：沿着问题理解 QKV，而不是从公式开始背诵。"
tags:
  - llm
  - attention
  - transformer
  - deep-learning
---

<p class="qkv-intro">读过《Attention Is All You Need》，也记得 softmax(QKᵀ / √dₖ)V，却仍然说不清：为什么偏偏是 Query、Key、Value？这篇文章补上公式之前的那段故事。</p>

许多 Transformer 教程会先给出三个矩阵，然后解释：Q 是“我想找什么”，K 是“我有什么”，V 是“实际内容”。这些比喻有帮助，但容易留下更深的疑问：既然每个 token 原本已经有一个向量，为什么还要把它变成三份？这些角色是怎样被需要出来的？

QKV 不是一条数学定理要求的唯一结构。它是围绕一个具体问题逐渐形成的设计：**怎样让神经网络根据当前需要，从一组信息里读出有用的内容？**

沿着这个问题往前走，三个字母就不再显得突然。

<div class="qkv-reading-map" markdown="1">
**阅读路线**

[固定编码的瓶颈](#bottleneck) → [按需读取与软对齐](#soft-alignment) → [为什么区分 K 和 V](#key-value) → [为什么分别学习 Q 和 K](#projections) → [self-attention 与矩阵公式](#self-attention)。

本文沿机器翻译与可微记忆读取这两条线展开，并非 attention 的完整历史。图中的权重和数值例子均为讲解构造，不是真实模型的测量结果。
</div>

<h2 id="bottleneck">先有一个瓶颈：整句话，只留一份摘要</h2>

假设我们要把“小猫坐在垫子上”翻译成“The cat sits on the mat”。早期一种典型的神经机器翻译方案是 encoder–decoder：encoder 用 RNN 或 LSTM 读完源句，把它编码成固定维度的表示；decoder 以这个表示为条件，逐步生成译文。Sutskever 等人在 2014 年的 Seq2Seq 工作是其中一个重要代表。[1]

为了抓住问题，暂时把传给 decoder 的状态统一记成一个句子编码 $c$。不论接下来要生成 cat、sits 还是 mat，源句信息都经过同一份编码。

<figure class="qkv-figure">
{% include blog/qkv-bottleneck.svg %}
<figcaption><span class="qkv-caption-label">图 1</span> 上：固定大小的句子编码是源句与译文之间的信息通道。下：保留各源位置的表示，根据当前解码状态生成不同的上下文向量 cₜ。为突出结构，图中省略了 RNN 的递归连接。</figcaption>
</figure>

这像是读完一段材料后，把原文收起来，只允许带着一张固定大小的笔记完成翻译。材料变长、细节变多，整理这张笔记就越来越困难。

这里不是在宣称“有限维实数向量在数学上绝不可能编码长句”。真正的问题是：在有限精度、有限模型能力和实际训练条件下，**让一个固定大小的表示承担所有细节，会形成学习与信息保留的瓶颈。** Bahdanau 等人正是以这一问题为出发点，提出了允许 decoder 直接读取源句各部分的机制。[2]

一个自然的改进是：既然 encoder 读到每个位置时都产生了隐藏表示，为什么只保留最终的摘要？何不把它们全部留下？

$$
h_1,h_2,\ldots,h_n
$$

$h_j$ 是原文第 $j$ 个位置的表示。在 Bahdanau 的双向 RNN encoder 中，它综合了两个方向的上下文；它不是一个孤立词的词典向量。[2]

保留了这些表示以后，decoder 就有机会在每一步回头查看原文。

<h2 id="soft-alignment">Attention：每一步，重新决定读哪里</h2>

问题从“怎样把全部内容压进一个摘要”变成了“这一刻，应该从原文读什么”。

翻译 cat 时，应该较多参考“小猫”；翻译 sits 时，应该较多参考“坐在”。我们希望得到类似下面的读取权重：

<div class="qkv-article-table" role="region" aria-label="示意翻译对齐权重" tabindex="0" markdown="1">

| 当前生成 | 小猫 | 坐在 | 垫子上 |
|---|---:|---:|---:|
| cat | **0.85** | 0.10 | 0.05 |
| sits | 0.10 | **0.80** | 0.10 |
| mat | 0.05 | 0.10 | **0.85** |

</div>

这不是人工给模型写的对齐规则，而是我们希望模型能够学会的一种行为。真实翻译也不会总有这么整齐的一对一关系：一个词可能需要参考多个位置。

怎样实现？令 decoder 在生成第 $t$ 个目标词之前的状态为 $s_{t-1}$。它概括了当前生成进度。先对每个源位置打分：

$$
e_{tj}=\operatorname{score}(s_{t-1},h_j).
$$

把分数转换成非负、总和为 1 的权重：

$$
\alpha_{tj}
=\frac{\exp(e_{tj})}{\sum_{r=1}^{n}\exp(e_{tr})}.
$$

再汇总原文表示：

$$
c_t=\sum_{j=1}^{n}\alpha_{tj}h_j.
$$

**这个 $c_t$ 就是“为当前生成步骤读出来的上下文”。** decoder 把它和自身状态等信息结合起来，预测目标词。到了下一步，状态变了，权重和上下文也可以随之改变。

Bahdanau 等人的工作在 2014 年发布预印本，发表于 ICLR 2015。其核心之一，是把翻译与这种软对齐放进同一个可训练模型。[2]

为什么是“软”的？因为它不必做出“只选第 3 个位置”的离散决定，而是允许以 0.1、0.8、0.1 的比例混合读取多个位置。分数、softmax、加权和组成连续的计算链，翻译损失可以沿着这条链反向传播。训练通常不需要额外告诉模型“这个目标词必须对齐那个源词”。

**到这里，attention 已经成立了，却还没有要求三个独立的 QKV 投影矩阵。**

## 打分函数，最初也不是 QKᵀ

Bahdanau attention 的打分函数常写为：

$$
e_{tj}=u_a^\top\tanh(W_s s_{t-1}+W_h h_j+b_a).
$$

它本质上是一个小神经网络：同时接收当前解码状态与候选源位置，输出一个标量。这通常被称为 additive attention。[2]

Luong 等人在 2015 年系统研究了其他打分形式，包括直接点积与带参数的双线性打分。[3] 为统一符号，可以写成：

$$
\operatorname{score}(s,h)=s^\top h
\qquad\text{或}\qquad
\operatorname{score}(s,h)=s^\top W h.
$$

这些形式都在回答同一件事：“当前状态与这个候选有多匹配？”

点积计算简单，也适合批量矩阵运算，但它不是 attention 的定义。**先有按需打分、再加权读取的结构，点积只是打分的一种实现。** 不要把一种后来非常成功的实现，误认为这个思想唯一可能的起点。

<h2 id="key-value">三个角色，其实已经藏在早期 attention 里</h2>

回看上面的计算，可以辨认出三种工作：

<div class="qkv-article-table" role="region" aria-label="早期 attention 与 QKV 的角色对应" tabindex="0" markdown="1">

| 工作 | 早期翻译模型中的对应物 | 通用名称 |
|---|---|---|
| 发起这次读取 | Decoder 状态 $s_{t-1}$ | Query |
| 供查询匹配、决定权重 | 源位置表示 $h_j$ | Key |
| 按权重读取的内容 | 同一个 $h_j$ | Value |

</div>

**最容易忽略的地方是：同一个 $h_j$，原本同时充当了 Key 和 Value。** 它既参与决定“要不要读这个位置”，又作为“读出的内容”被加权汇总。

但这两件事为什么一定要用同一套表示？

想象我们要查询联系人。查询是“财务部的张三”；为了找到对应的人，我们可能需要姓名、部门等字段；找到以后，真正想读取的却是电话号码。用于定位的字段和需要返回的内容，并不相同。

神经网络也可以把这两种功能分开：匹配时突出有利于定位的信息，读取时保留有利于后续计算的信息。

这不只是事后的类比。2015 年的 End-to-End Memory Networks 已使用不同的输入记忆与输出记忆表示；2016 年的 Key-Value Memory Networks 更明确地把寻址和读取阶段的编码分开。[4][5] 这是一条与机器翻译 attention 相关的研究线，而不是说所有论文构成了唯一、严格串行的继承链。

于是，一次通用读取可以写成：

$$
e_j=\operatorname{score}(q,k_j),\qquad
\alpha_j=\operatorname{softmax}(e)_j,\qquad
o=\sum_j\alpha_jv_j.
$$

<figure class="qkv-figure">
{% include blog/qkv-read.svg %}
<figcaption><span class="qkv-caption-label">图 2</span> Key 与 Value 按位置一一对应：对 k₂ 的高匹配分数，最终使 v₂ 获得更大权重。图中的 0.1、0.8、0.1 是人为构造的示意值。</figcaption>
</figure>

在这个层面上，QKV 是三个**计算角色**，不一定意味着三份毫无关联的原始数据。Key 与 Value 可以由同一份输入产生，也可以相同；Query 则表示这一次读取的条件。

这和字典也有区别。普通字典通常按精确 key 返回一项；attention 用连续的兼容分数，对一组 Value 做软读取。

<h2 id="projections">为什么还需要 WQ、WK、WV？</h2>

下面暂时离开历史，做一个设计推演。它用来解释结构为什么合理，不是在复述研究者当年的全部思考过程。

假设每个位置已有一个行向量 $x_i$。最省事的方案是直接令：

$$
q_i=k_i=v_i=x_i.
$$

这样依然可以匹配和汇总。**三个独立投影并不是 attention 成立的数学必要条件。** 但让同一个向量承担所有角色，会带来限制。

### 相关，不等于相似

考虑“小猫吃了鱼”。在理解“吃”的时候，可能需要知道谁在吃、吃了什么。动作与参与者之间有关系，却未必应该在原始表示空间里非常相似。

我们真正要学的是“某个位置发起的需求，是否与另一个位置兼容”，而不只是两个词有多像。

因此，把查询方与候选方分别变换：

$$
q_i=x_iW_Q,\qquad k_j=x_jW_K.
$$

它们的点积等于：

$$
q_i k_j^\top
=x_iW_QW_K^\top x_j^\top.
$$

这相当于学习了一个由 $W_QW_K^\top$ 参数化的双线性关系。在较小的 head 维度下，它还有相应的秩限制；这里不需要把它当成任意、无限表达能力的打分器。

分开投影的另一个作用是允许**方向性**：

$$
\operatorname{score}(i,j)\ne\operatorname{score}(j,i).
$$

“动作寻找施事者”和“实体寻找相关动作”可以是两种不同的关系。如果直接令 $Q=K=X$，未加 mask 的原始点积矩阵 $XX^\top$ 就是对称的；不过即便如此，逐行 softmax 后的权重矩阵也不一定对称。

### 用来匹配的特征，不一定适合直接传递

再令：

$$
v_j=x_jW_V.
$$

$W_V$ 让模型另外学习“选中这个位置后，应该传递什么表示”，不必让全部内容都服从匹配空间的要求。Value 的维度也不必与 Query、Key 相同；需要相同维度的是参与点积的 Q 与 K。

因此，可以这样读三组投影：

- $W_Q$：把当前位置转换成一次查询。
- $W_K$：把候选位置转换成供匹配的表示。
- $W_V$：把候选位置转换成待传递的内容。

这些是计算上的分工，不是人工制定的语义标签。我们没有规定某个维度必须表示“主语”，也没有保证一个 head 专门负责指代。参数通过任务损失共同训练，具体学出的特征可能很分散。

## 手算一次：匹配与内容到底怎样分开？

用一个二维例子把过程走完。设当前查询为：

$$
q=[1,0].
$$

有三个候选位置：

<div class="qkv-article-table" role="region" aria-label="手算示例的 Key 和 Value" tabindex="0" markdown="1">

| 位置 | Key | Value |
|---|---|---|
| A | $[2,0]$ | $[1,0]$ |
| B | $[0,2]$ | $[0,1]$ |
| C | $[1,1]$ | $[1,1]$ |

</div>

先计算点积，得到 $[2,0,1]$。如果使用 Transformer 的缩放，$d_k=2$，所以送入 softmax 的分数是：

$$
\frac{[2,0,1]}{\sqrt{2}}
\approx[1.414,0,0.707].
$$

softmax 后的权重约为：

$$
\alpha\approx[0.576,0.140,0.284].
$$

最后汇总 **Value**：

$$
\begin{aligned}
o&=0.576[1,0]+0.140[0,1]+0.284[1,1]\\
 &\approx[0.860,0.424].
\end{aligned}
$$

这个例子里，Key A 最匹配，所以 Value A 获得最大的权重。但是输出既不是 Key A，也不是简单返回 Value A，而是三个 Value 的加权和。

现在做两个思想实验：保持 Q、K 不变，只改 V，权重保持不变，但读出的内容会变；保持 V 不变，只改 Q 或 K，内容本身没变，但混合比例会变。**“匹配机制”和“内容通道”的分工，在这里变得非常具体。**

<h2 id="self-attention">Self-attention：让同一句话内部相互读取</h2>

到目前为止，查询者来自译文 decoder，被读取的信息来自源句 encoder。两者来源不同，很容易理解。

再往前一步：为什么同一句话中的每个位置，不能向同一句话的其他位置发起查询？

例如“小猫吃完鱼后，它睡着了”。“它”这个位置如果能读取“小猫”及周围上下文的信息，就有机会形成更有用的表示。这里描述的是模型可学习的行为，不是在断言某个真实 head 必然承担这一功能。

令上一层所有位置的表示构成矩阵 $X$。同一份输入经过三组投影：

$$
Q=XW_Q,\qquad K=XW_K,\qquad V=XW_V.
$$

<figure class="qkv-figure">
{% include blog/qkv-self.svg %}
<figcaption><span class="qkv-caption-label">图 3</span> Self-attention 的“self”说的是查询与记忆来自同一组位置。它不代表只看自己，也不代表 Q、K、V 必须相等。图中画的是一个 head；残差、输出投影和其他子层暂时省略。</figcaption>
</figure>

这与 cross-attention 的区别在于来源。后者可以用一组表示 $X$ 产生 Q，用另一组表示 $H$ 产生 K、V：

$$
Q=XW_Q,\qquad K=HW_K,\qquad V=HW_V.
$$

Transformer 原论文中，encoder–decoder attention 就采用这种分工；self-attention 则从同一序列的上一层表示中产生三者。[6]

还有一个容易混淆的细节：同一层的各位置基于这一层输入一起计算，而不是“先更新第一个词，再拿更新后的结果算第二个词”。这使许多位置的计算可以并行进行。

## 矩阵公式，是把所有查询一起写下来

假设序列有 $n$ 个位置，输入维度为 $d$。对于一个 attention head：

$$
\begin{aligned}
X&\in\mathbb{R}^{n\times d},\\
Q,K&\in\mathbb{R}^{n\times d_k},\\
V&\in\mathbb{R}^{n\times d_v}.
\end{aligned}
$$

于是 $QK^\top$ 是一张 $n\times n$ 的打分表：第 $i$ 行是第 $i$ 个位置发起的查询，第 $j$ 列是候选位置 $j$。

对每行分别做 softmax，再乘以 V：

<div class="qkv-formula">
$$
\underbrace{A}_{n\times n}
=\operatorname{softmax}_{\text{row}}
\left(\frac{QK^\top}{\sqrt{d_k}}\right),
\qquad
\underbrace{O}_{n\times d_v}=AV.
$$
</div>

这里的 $A$ 是读取权重，不是词表上的下一个 token 概率。$O$ 则是每个位置在本 head 里读到的向量，后面还有输出投影、残差、前馈网络等计算。

为什么除以 $\sqrt{d_k}$？在各分量独立、均值为 0、方差为 1 的简化假设下，$q\cdot k$ 的方差随 $d_k$ 增长。缩放能控制分数尺度，避免 softmax 太容易进入梯度很小的饱和区域。这是 Transformer 原论文给出的动机，不是说训练后的向量永远满足该独立分布假设。[6]

多头也可以沿用相同理解：每个 head 各有投影和读取权重，独立汇总，再拼接并投影回模型维度。它让多个读取模式同时存在，而不是把所有关系都挤进同一组混合权重里。不同 head 可以分工，但没有“一个 head 必然对应一种语法规则”的保证。[6]

## 从 attention，到 Transformer

QKV 解释了读取操作，但还不是完整的序列模型。

只有内容匹配，没有位置信息，attention 不会自动知道一句话原本的词序；位置编码等机制负责提供顺序信息。语言生成还需要因果 mask，禁止当前位置读取未来位置。残差连接、归一化、前馈网络等组件，则帮助构成可训练的深层模型。

在早期翻译系统里，RNN 负责构造序列表示，attention 辅助 decoder 读取源句。2017 年的 Transformer 把 self-attention 放到跨位置交互的中心，构建了不依赖循环与卷积的翻译架构。它并非 attention 或 self-attention 的起点，而是一次重要的架构整合。[6]

回头看，三个字母背后是一连串具体的需要：先保留原文各位置，再按当前状态读取；随后把“怎样找到信息”与“读出什么内容”分开；最后让同一句话内部的每个位置都能发起这种读取。

公式不再是起点，而是这些选择的紧凑写法。

## 附记：这与 KV cache 有什么关系？

在因果 self-attention 的逐 token 推理中，新位置产生自己的 Q、K、V，再用新 Q 查询可见位置的 K、V。旧位置看不到后来追加的 token，因此在模型与已有前缀不变时，它们在各层算出的 K、V 可以保留复用。

这就是 KV cache：保存以后还要读取的中间结果，避免反复计算历史位置。旧 Q 通常不必保存，因为下一次读取由新位置的 Q 发起。它是推理加速机制；理解 QKV 本身，先把“查询、匹配、读取”这三件事分清就足够了。

## 参考文献

<div class="qkv-references" markdown="1">

1. Sutskever, Vinyals & Le. [Sequence to Sequence Learning with Neural Networks](https://arxiv.org/abs/1409.3215). 2014. 固定长度编码与 LSTM encoder–decoder 的代表工作。
2. Bahdanau, Cho & Bengio. [Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473). 预印本 2014，ICLR 2015. 理解软对齐与 additive attention 的起点。
3. Luong, Pham & Manning. [Effective Approaches to Attention-based Neural Machine Translation](https://aclanthology.org/D15-1166/). EMNLP 2015. 包括 dot、general 等打分形式。
4. Sukhbaatar et al. [End-To-End Memory Networks](https://arxiv.org/abs/1503.08895). NeurIPS 2015. 可微记忆读取及不同的输入、输出记忆表示。
5. Miller et al. [Key-Value Memory Networks for Directly Reading Documents](https://arxiv.org/abs/1606.03126). EMNLP 2016. 明确区分寻址和读取的表示。
6. Vaswani et al. [Attention Is All You Need](https://arxiv.org/abs/1706.03762). NeurIPS 2017. Scaled dot-product、multi-head attention 与 Transformer 架构。

</div>
