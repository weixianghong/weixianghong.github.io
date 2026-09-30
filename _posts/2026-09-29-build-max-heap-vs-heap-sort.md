---
title: "为什么 Build-Max-Heap 是 O(n)，而 Heap Sort 却是 O(n log n)？"
date: 2026-09-29
permalink: /blogs/build-max-heap-vs-heap-sort/
redirect_from:
  - /build-max-heap-vs-heap-sort/
author_profile: true
read_time: true
excerpt: "两个过程都在反复调用 O(log n) 的 Max-Heapify，为什么建堆是 O(n)，堆排序却仍然是 O(n log n)？关键不在单次最坏复杂度，而在昂贵操作如何分布。"
tags:
  - algorithms
  - data-structures
  - heap
  - sorting
  - complexity-analysis
---

**目录**
* [先把两个疑问摆出来](#先把两个疑问摆出来)
* [Build-Max-Heap 是 O(n)，对 Heap Sort 本身重要吗？](#build-max-heap-是-on对-heap-sort-本身重要吗)
* [为什么 Build-Max-Heap 不是 O(n log n)？](#为什么-build-max-heap-不是-on-log-n)
* [为什么 Heap Sort 没有出现同样的“神奇现象”？](#为什么-heap-sort-没有出现同样的神奇现象)
* [只看前一半，就已经足够看出问题](#只看前一半就已经足够看出问题)
* [把排序阶段真的加起来](#把排序阶段真的加起来)
* [两者最本质的区别](#两者最本质的区别)
* [一个复杂度直觉图](#一个复杂度直觉图)
* [最后总结](#最后总结)

---

Heap Sort 的复杂度分析里，有一个非常值得停下来想一想的地方。

我们通常知道：

- Max-Heapify：最坏 $O(\log n)$
- Build-Max-Heap：$O(n)$
- Heap Sort：$O(n\log n)$

这些结论很容易背下来，但如果认真追问，会立刻冒出两个问题：

1. **既然后面的 Heap Sort 本来就要 $O(n\log n)$，为什么教材还要花力气证明 Build-Max-Heap 是 $O(n)$，而不是 $O(n\log n)$？**
2. **Heap Sort 在不断把最大元素扔到数组尾部，heap size 明明越来越小，为什么总复杂度没有像 Build-Max-Heap 一样，从“看似 $O(n\log n)$”神奇地降成 $O(n)$？**

这两个问题看起来分开，实际上都在问同一件事：

> **当一个算法反复调用某个最坏为 $O(\log n)$ 的操作时，我们到底能不能直接写成“调用次数 × $O(\log n)$”？**

答案是：**有时候可以，有时候会非常松。关键要看这些操作的真实成本是怎么分布的。**

---

## Build-Max-Heap 是 O(n)，对 Heap Sort 本身重要吗？

如果我们的目标只是回答：

> Heap Sort 最终属于哪个渐近复杂度？

那么确实，**Build-Max-Heap 是 $O(n)$ 还是 $O(n\log n)$，并不会改变最终答案。**

因为即使建堆真的是：

$$
O(n\log n),
$$

再加上后面的排序阶段：

$$
O(n\log n),
$$

总时间仍然是：

$$
O(n\log n)+O(n\log n)
=
O(n\log n).
$$

真实情况只是：

$$
O(n)+O(n\log n)
=
O(n\log n).
$$

所以，如果只为了证明 Heap Sort 的最终 big-O，确实没有必要把 Build-Max-Heap 的线性复杂度研究得那么细。

但这个结论仍然值得专门证明，原因至少有三个。

### 1. Build-Max-Heap 本身就是一个重要算法

如果我们已经有 $n$ 个元素，要把它们批量变成一个 heap：

- 一个一个 insert：最坏会做到 $O(n\log n)$
- bottom-up Build-Max-Heap：可以做到 $O(n)$

也就是说，**“批量建堆”本身就比“逐个插入”更快。**

### 2. 它是复杂度分析里一个非常经典的反例

一个常见的错误思路是：

> 有 $O(n)$ 个节点，每次 Heapify 最坏 $O(\log n)$，所以总共 $O(n\log n)$。

这个上界不能说错，但它很松，因为它偷偷假设：

> 每一次 Heapify 都有机会走完整棵树的高度。

实际上完全不是这样。

### 3. 它告诉我们 Heap Sort 真正贵在哪里

Heap Sort 的 $O(n\log n)$ 不是花在初始化上，而是花在后面反复：

> extract max → 修复 root → extract max → 修复 root → …

所以 Build-Max-Heap 的 $O(n)$ 虽然不改变 Heap Sort 的最终复杂度等级，却揭示了 heap 的一个非常重要的结构性质。

---

## 为什么 Build-Max-Heap 不是 O(n log n)？

核心原因只有一句：

> **越靠近树底部的节点越多，而越靠近底部，Heapify 能向下走的距离就越短。**

一个 heap 是一棵 nearly complete binary tree。

如果从“离叶子的高度”来看：

- 大约 $n/2$ 个节点是 leaf，向下走 **0 层**
- 大约 $n/4$ 个节点最多向下走 **1 层**
- 大约 $n/8$ 个节点最多向下走 **2 层**
- 大约 $n/16$ 个节点最多向下走 **3 层**
- …
- 真正能走 $O(\log n)$ 层的节点，只有根附近极少数几个

因此总工作量不是：

$$
n\cdot\log n,
$$

而更像：

$$
T(n)
\le
\frac{n}{4}\cdot 1
+
\frac{n}{8}\cdot 2
+
\frac{n}{16}\cdot 3
+\cdots
$$

把 $n$ 提出来：

$$
T(n)
\le
n
\left(
\frac14
+
\frac{2}{8}
+
\frac{3}{16}
+\cdots
\right).
$$

括号里的级数是收敛的：

$$
\frac14+\frac28+\frac3{16}+\cdots
=
\sum_{h=1}^{\infty}\frac{h}{2^{h+1}}
=
1.
$$

因此：

$$
T(n)=O(n).
$$

如果想写得更严格一些，可以使用这样的事实：

> 高度为 $h$ 的节点数量至多约为
>
> $$
> \left\lceil \frac{n}{2^{h+1}}\right\rceil.
> $$

而一个高度为 $h$ 的节点做 Heapify，最多向下走 $h$ 层。

于是：

$$
T(n)
\le
\sum_{h=0}^{\lfloor\log n\rfloor}
\left\lceil\frac{n}{2^{h+1}}\right\rceil O(h)
=
O(n).
$$

但比公式更重要的是下面这个直觉：

> **Build-Max-Heap 里，“昂贵的 Heapify”非常稀少。**

能走很远的节点只有少数；数量最多的那些节点，本身几乎不用走。

这就是它从“看似 $O(n\log n)$”降到 $O(n)$ 的根本原因。

---

## 为什么 Heap Sort 没有出现同样的“神奇现象”？

Heap Sort 在建好 max heap 以后，会反复做：

1. 把 root（最大值）和当前 heap 的最后一个元素交换
2. 把 heap size 减 1
3. 对 root 做 Max-Heapify

所以 heap size 的确在不断变小：

$$
n,\ n-1,\ n-2,\ \dots,\ 2.
$$

这时候一个非常自然的想法是：

> 既然问题规模越来越小，那总复杂度会不会也像 Build-Max-Heap 一样比 $n\log n$ 小很多？

真正应该看的总成本是：

$$
\log n
+
\log(n-1)
+
\log(n-2)
+\cdots+
\log 2.
$$

确实不是每一步都等于 $\log n$。

但问题在于：

> **$\log n$ 下降得太慢了。**

例如：

$$
n=2^{20}=1{,}048{,}576.
$$

heap 高度大约是：

$$
20.
$$

即使已经删掉一半元素，只剩：

$$
2^{19}=524{,}288,
$$

高度也只是从：

$$
20
\rightarrow
19.
$$

也就是说：

> **数据量已经减少了 50%，树高却只少了 1 层。**

这就是为什么“heap 在缩小”不足以让整个算法掉一个复杂度等级。

---

## 只看前一半，就已经足够看出问题

这是理解 Heap Sort 为什么仍然是 $O(n\log n)$ 的最直观办法。

我们甚至不需要分析完整个排序过程。

只看 heap size 从：

$$
n
$$

缩小到：

$$
\frac n2
$$

这一段。

这一段已经包含大约：

$$
\frac n2
$$

次 extraction。

而这整个阶段里，heap size 始终至少是：

$$
\frac n2.
$$

因此 heap 的高度仍然是：

$$
\log\frac n2
=
\log n-1
=
\Theta(\log n).
$$

换句话说，**光前一半，就有 $\Theta(n)$ 次从 root 开始的修复，而这些 heap 仍然有 $\Theta(\log n)$ 的高度。**

从“潜在工作规模”的直觉看：

$$
\frac n2\log\frac n2
=
\frac n2(\log n-1)
=
\Theta(n\log n).
$$

所以 heap 虽然在缩小，但它缩小得不够快。

> **“问题规模在下降”不等于“总复杂度一定降低一个等级”。**
>
> 关键是：它以什么速度下降，以及有多少次操作仍然发生在“大规模”阶段。

这里还要注意一个严谨性问题：

上面的“前一半”论证很好地解释了为什么 $O(n\log n)$ 这个上界不会因为 heap 缩小就自动变成 $O(n)$，但它本身并不是在证明每一次 Heapify 都一定真的走满 $\Theta(\log n)$ 层。

要得到 Heap Sort **worst-case 的紧确界**，我们还可以结合 comparison sort 的下界：

$$
\Omega(n\log n).
$$

而 Heap Sort 已经有：

$$
O(n\log n)
$$

的上界，所以：

$$
\boxed{
\text{Heap Sort worst case}
=
\Theta(n\log n)
}
$$

---

## 把排序阶段真的加起来

如果想更直接地看这个总和：

$$
T(n)
=
\sum_{k=2}^{n}\log k.
$$

利用：

$$
\log a+\log b=\log(ab),
$$

可以得到：

$$
T(n)
=
\log(2\cdot3\cdots n)
=
\log(n!).
$$

而由 Stirling approximation：

$$
\log(n!)
=
\Theta(n\log n).
$$

因此：

$$
\sum_{k=2}^{n}\log k
=
\Theta(n\log n).
$$

所以“规模逐步从 $n$ 降到 1”并没有把这个和变成 $O(n)$。

---

## 两者最本质的区别

现在可以把两个过程放在一起看。

### Build-Max-Heap

Heapify 从**树的不同位置**开始：

- 大量节点在叶子附近
- 很多操作只可能走 0、1、2 层
- 越能走得深的节点，数量越少
- 昂贵操作的数量呈指数级减少

所以：

$$
\frac n4\cdot1
+
\frac n8\cdot2
+
\frac n{16}\cdot3
+\cdots
=
O(n).
$$

### Heap Sort

每一次删除最大值以后，都重新从：

$$
\boxed{\text{root}}
$$

开始修复。

也就是：

$$
\text{root},
\text{root},
\text{root},
\text{root},
\dots
$$

尤其在前面很长一段时间里，heap 的高度一直还是：

$$
\Theta(\log n).
$$

因此不是：

> 很多便宜操作 + 极少数昂贵操作

而更接近：

> **大量操作都从最昂贵的位置开始。**

可以把区别压缩成这张表：

| | Build-Max-Heap | Heap Sort extraction |
|---|---|---|
| Heapify 从哪里开始？ | 树中很多不同节点 | 一遍又一遍从 root |
| 大量操作贵吗？ | 不贵，大部分节点靠近叶子 | 很长一段时间都可能很贵 |
| 昂贵操作的数量 | 指数级减少 | 线性数量的操作发生在仍然很高的 heap 上 |
| 总复杂度 | $O(n)$ | $O(n\log n)$ |

我认为最值得记住的一句话是：

> **Build-Max-Heap 之所以便宜，是因为昂贵的 Heapify 很少；Heap Sort 之所以贵，是因为它反复从最昂贵的位置——root——开始 Heapify。**

---

## 一个复杂度直觉图

下面这个小图不是在描述某台机器上的真实运行时间，只是用一个固定的 $n$ 来帮助建立增长速度的直觉。

<div style="border:1px solid #d0d7de;border-radius:12px;padding:18px 20px;margin:24px 0;background:#fafbfc;">
  <div style="font-size:1.05em;font-weight:700;margin-bottom:6px;">当 n = 1024 时，不同复杂度的大致操作规模</div>
  <div style="font-size:0.9em;color:#57606a;margin-bottom:16px;">假设 log 以 2 为底。条形长度使用对数尺度，仅用于视觉比较。</div>

  <div style="display:grid;grid-template-columns:110px 120px 1fr;gap:10px 14px;align-items:center;">
    <div><strong>O(1)</strong></div>
    <div>1</div>
    <div style="height:14px;background:#d0d7de;border-radius:7px;width:6%;"></div>

    <div><strong>O(log n)</strong></div>
    <div>10</div>
    <div style="height:14px;background:#afb8c1;border-radius:7px;width:18%;"></div>

    <div><strong>O(n)</strong></div>
    <div>1,024</div>
    <div style="height:14px;background:#8c959f;border-radius:7px;width:43%;"></div>

    <div><strong>O(n log n)</strong></div>
    <div>10,240</div>
    <div style="height:14px;background:#6e7781;border-radius:7px;width:58%;"></div>

    <div><strong>O(n²)</strong></div>
    <div>1,048,576</div>
    <div style="height:14px;background:#57606a;border-radius:7px;width:86%;"></div>
  </div>
</div>

对本文最重要的是中间这两项：

$$
O(n)
\quad\text{vs.}\quad
O(n\log n).
$$

当 $n$ 很小时，两者看起来差别不大；但随着 $n$ 增长，那个额外的 $\log n$ 会持续放大差距。

不过 Build-Max-Heap 能做到 $O(n)$，并不是因为单次 Heapify 变成了 $O(1)$，而是因为：

> **绝大多数 Heapify 根本没有机会走到 $\log n$ 那么深。**

这也是本文最重要的复杂度分析思路。

---

## 最后总结

这两个算法都使用同一个原语：

$$
\text{Max-Heapify}.
$$

单次 Max-Heapify 最坏都是：

$$
O(\log n).
$$

但总复杂度完全不同：

$$
\boxed{
\text{Build-Max-Heap}=O(n)
}
$$

$$
\boxed{
\text{Heap Sort}=\Theta(n\log n)
}
$$

原因不是 Heapify 本身发生了变化，而是**昂贵操作的分布不同**。

Build-Max-Heap：

> 越昂贵的 Heapify，出现得越少。

Heap Sort：

> 一遍又一遍从 root 开始修复，而且在很长一段时间里 heap 仍然很高。

所以复杂度分析真正该问的，不是：

> “最坏一次多少钱？”

而是：

> **“整个算法里，不同价格的操作分别发生了多少次？”**

这也是为什么不能机械地看到 $n$ 次操作、每次最坏 $O(\log n)$，就立刻认为真正的总成本一定是 $n\log n$。
