---
title: "Why Build-Max-Heap Is O(n), but Heap Sort Is O(n log n)"
date: 2026-09-29
permalink: /build-max-heap-vs-heap-sort/
author_profile: true
read_time: true
excerpt: "Two heap procedures look like n applications of an O(log n) operation. Why is bottom-up heap construction linear, while Heap Sort still takes O(n log n)?"
tags:
  - algorithms
  - data-structures
  - heap
  - sorting
  - complexity-analysis
---

**Table of Contents**
* [Two Questions That Look Almost the Same](#two-questions-that-look-almost-the-same)
* [Does the O(n) Build-Heap Bound Actually Matter for Heap Sort?](#does-the-on-build-heap-bound-actually-matter-for-heap-sort)
* [Why Build-Max-Heap Is O(n)](#why-build-max-heap-is-on)
* [Why the Sorting Phase Does Not Collapse to O(n)](#why-the-sorting-phase-does-not-collapse-to-on)
* [A Simple Lower-Bound Argument: Just Look at the First Half](#a-simple-lower-bound-argument-just-look-at-the-first-half)
* [The Exact Shape of the Sum](#the-exact-shape-of-the-sum)
* [Why Build-Heap Gets the "Magic" but Heap Sort Does Not](#why-build-heap-gets-the-magic-but-heap-sort-does-not)
* [A Useful Mental Model](#a-useful-mental-model)
* [Takeaway](#takeaway)

---

Heap Sort contains a small complexity-analysis puzzle that is easy to memorize and surprisingly easy to misunderstand.

A standard implementation has two stages:

1. Turn an unordered array into a max heap.
2. Repeatedly move the maximum element to the end of the array and restore the heap.

The first stage uses **Build-Max-Heap**. The second repeatedly uses **Max-Heapify**.

Since a single Max-Heapify can take (O(log n)), a first guess might be:

$$
	ext{Build-Max-Heap} stackrel{?}{=} O(nlog n)
$$

But the real bound is:

$$
	ext{Build-Max-Heap} = O(n)
$$

Then another question appears. During Heap Sort, the heap keeps shrinking. If the problem size is getting smaller after every extraction, why does the sorting phase remain (O(nlog n))? Why does it not show the same "surprising" linear behavior as Build-Max-Heap?

Those are the two questions this post is about.

## Two Questions That Look Almost the Same

At a superficial level, the two phases look very similar.

For heap construction, we call Max-Heapify on many nodes:

$$
	ext{Max-Heapify}(A, lfloor n/2 floor),
	ext{Max-Heapify}(A, lfloor n/2 floor - 1),
dots,
	ext{Max-Heapify}(A, 1)
$$

For sorting, we repeatedly call Max-Heapify on the root while the heap size decreases:

$$
n, n-1, n-2, dots, 2
$$

Since Max-Heapify is often summarized as an (O(log n)) operation, both phases can look like

$$
O(n) 	imes O(log n).
$$

That multiplication is safe as a loose upper bound, but it hides the important question:

> **How many expensive Heapify operations are there, and how expensive is each one actually allowed to be?**

That distribution of work is the whole story.

## Does the O(n) Build-Heap Bound Actually Matter for Heap Sort?

If all we want is the final asymptotic complexity class of Heap Sort, then surprisingly, **not that much**.

Even if Build-Max-Heap were (O(nlog n)), we would still have:

$$
O(nlog n) + O(nlog n)
=
O(nlog n).
$$

The real algorithm is better:

$$
O(n) + O(nlog n)
=
O(nlog n),
$$

but the final big-O category does not change.

So why do textbooks spend time proving that Build-Max-Heap is linear?

There are several good reasons.

First, **Build-Max-Heap is an important algorithm on its own**. If we already have (n) elements and want to turn them into a heap, bottom-up construction takes (O(n)). Inserting the same (n) elements one by one can take (O(nlog n)).

Second, it is a classic lesson in complexity analysis:

> You cannot always take the worst-case cost of one operation and multiply it by the number of operations.

The individual operations may have very different costs.

Third, it tells us where Heap Sort really spends its time. The expensive part is not initialization. The (O(nlog n)) cost comes from repeatedly extracting the maximum and repairing the root.

So the linear-time build is not what makes Heap Sort asymptotically better than (O(nlog n)), but it is still a fundamental property of heaps.

## Why Build-Max-Heap Is O(n)

The key fact is that a heap is a nearly complete binary tree.

Most nodes are near the bottom.

For a 1-indexed heap:

- nodes (lfloor n/2 floor + 1,dots,n) are leaves;
- leaves need no Heapify work at all;
- nodes one level above the leaves can move down at most one level;
- nodes two levels above can move down at most two levels;
- only a tiny number of nodes near the root can move down many levels.

Ignoring floors and ceilings for intuition, the distribution looks like this:

| Height above the leaves | Number of nodes | Max downward work per node |
|---:|---:|---:|
| 0 | (approx n/2) | 0 |
| 1 | (approx n/4) | 1 |
| 2 | (approx n/8) | 2 |
| 3 | (approx n/16) | 3 |
| ... | ... | ... |

Therefore the total work is bounded by something like

$$
T(n)
le
rac{n}{4}(1)
+
rac{n}{8}(2)
+
rac{n}{16}(3)
+cdots
$$

Factor out (n):

$$
T(n)
le
n
left(
rac{1}{4}
+
rac{2}{8}
+
rac{3}{16}
+cdots
ight).
$$

The infinite series converges to a constant, so:

$$
T(n)=O(n).
$$

The important idea is more useful than the algebra:

> **The Heapify operations that are capable of traveling far down the tree are exponentially rare.**

There is only one root. There are only a few nodes near the root. The huge majority of nodes are near the leaves and can move only a tiny distance.

That is where the linear-time bound comes from.

## Why the Sorting Phase Does Not Collapse to O(n)

Now consider the sorting phase.

After building the max heap, Heap Sort repeatedly:

1. swaps the root with the last element in the current heap,
2. reduces the heap size by one,
3. calls Max-Heapify on the root.

The heap sizes are:

$$
n, n-1, n-2,dots,2.
$$

So a more accurate upper-bound expression is not simply (nlog n). It is:

$$
log n
+
log(n-1)
+
log(n-2)
+cdots+
log 2.
$$

At first glance, this looks promising. The heap is shrinking after every iteration.

But the crucial point is that **logarithms shrink very slowly**.

Suppose:

$$
n=2^{20}=1{,}048{,}576.
$$

The heap height is about 20.

After we have already removed half the elements, the heap size is:

$$
2^{19}=524{,}288,
$$

and the height is still about 19.

We removed **50% of the data**, but the maximum Heapify depth dropped by only **one level**.

That is the opposite of what happens during Build-Max-Heap. During heap construction, most operations start near the leaves. During sorting, we keep returning to the root — the most expensive place to start.

## A Simple Lower-Bound Argument: Just Look at the First Half

There is a particularly clean way to see why the shrinking heap does not save us.

Ignore the second half of the algorithm completely.

Look only at the period while the heap size decreases from (n) to (n/2).

That already contains roughly:

$$
rac{n}{2}
$$

iterations.

During all of those iterations, the heap has at least (n/2) elements, so its height is on the order of:

$$
lograc{n}{2}
=
log n - 1
=
Theta(log n).
$$

Therefore, the first half alone still consists of (Theta(n)) root repairs on heaps whose height is (Theta(log n)).

As an intuition for the scale of the work:

$$
rac{n}{2}lograc{n}{2}
=
rac{n}{2}(log n - 1)
=
Theta(nlog n).
$$

So the fact that the heap eventually becomes small does not matter enough. It becomes small **too late** to change the asymptotic order.

A useful general lesson is:

> **"The problem size is decreasing" does not automatically mean the total complexity drops by a whole asymptotic class. The rate of decrease matters.**

## The Exact Shape of the Sum

We can also analyze the sorting phase as a sum:

$$
sum_{k=2}^{n}log k.
$$

Using the identity

$$
log a + log b = log(ab),
$$

we get:

$$
sum_{k=2}^{n}log k
=
log(2cdot3cdots n)
=
log(n!).
$$

And by Stirling's approximation,

$$
log(n!)
=
Theta(nlog n).
$$

Therefore the natural upper bound for the repeated root Heapify work is:

$$
O(nlog n).
$$

Heap Sort is a comparison-based sorting algorithm. The comparison-sorting decision-tree lower bound tells us that any comparison sort requires

$$
Omega(nlog n)
$$

comparisons in the worst case.

Combining the upper and lower bounds gives the familiar tight worst-case result:

$$
oxed{	ext{Heap Sort}=Theta(nlog n)}
$$

## Why Build-Heap Gets the "Magic" but Heap Sort Does Not

The cleanest way to compare the two algorithms is to ask:

> **Where do the Heapify calls start?**

For **Build-Max-Heap**, they start all over the tree.

Most start close to the leaves.

The work distribution is roughly:

$$
rac{n}{4}(1)
+
rac{n}{8}(2)
+
rac{n}{16}(3)
+cdots
$$

The farther an operation can travel, the fewer such operations exist.

For **Heap Sort**, the repair repeatedly starts at the root:

$$
	ext{root},
	ext{root},
	ext{root},
	ext{root},
dots
$$

And for a large fraction of the algorithm, the heap is still large enough that the root has (Theta(log n)) levels beneath it.

So the two situations are fundamentally different:

| Phase | Where Heapify starts | Distribution of expensive operations | Total |
|---|---|---|---:|
| Build-Max-Heap | Many different nodes | Deep Heapify calls are exponentially rare | (O(n)) |
| Heap Sort extraction | Root again and again | A linear number of calls occur while the heap is still tall | (O(nlog n)) |

This is the distinction that the shorthand "(n) operations times (O(log n))" fails to capture.

## A Useful Mental Model

I find the following sentence the easiest one to remember:

> **Build-Max-Heap is cheap because expensive Heapify calls are rare; Heap Sort is expensive because it repeatedly Heapifies from the most expensive location.**

There is also a connection to priority queues.

A max heap gives us:

$$
	ext{getMax}=O(1),
$$

because the maximum is always stored at the root.

But after removing that maximum, restoring the heap may take:

$$
O(log n).
$$

So repeatedly deleting the maximum naturally produces:

$$
n 	imes O(log n)
=
O(nlog n)
$$

overall.

This is a useful reminder that **looking at the minimum or maximum can be cheap, while deleting it and maintaining the data structure's invariant is where the real cost appears**.

## Takeaway

The surprising part of heap analysis is not the formulas themselves. It is the distribution of work.

For Build-Max-Heap:

- many nodes require no work;
- many more require only one or two levels of work;
- very few nodes can trigger a long Heapify;
- the weighted sum converges to (O(n)).

For Heap Sort:

- the heap size does shrink;
- but logarithmic height shrinks very slowly;
- we repeatedly repair from the root;
- even the first half of the extraction process happens on heaps of height (Theta(log n));
- therefore the sorting phase remains (O(nlog n)).

So the real contrast is:

$$
oxed{
	ext{Build-Max-Heap: expensive operations are rare}
}
$$

versus

$$
oxed{
	ext{Heap Sort: expensive root repairs happen repeatedly}
}
$$

That is why two procedures built from the same Max-Heapify primitive end up with different total complexities.
