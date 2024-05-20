---
linktitle: "ST表"
title: "ST表"
type: docs
math: true
date: 2023-04-21

menu:
    ps-algorithms:
        parent: Contents
        weight: 6
---

> 已知一个数列 $a_1, \cdots, a_n$。给定 $l, r$，问区间 $a_l, \cdots a_r$ 的某种值。

ST表是一种处理上述格式的区间查询问题的利器。这类问题通常被称为 RMQ (range minimum query)，但实际上 ST 表可以应对的问题远不止最小值这一种。

大家已经或多或少地接触过一些区间查询问题，例如求区间的和。以 $+$, $xor$ (异或操作) 这类算符为代表的操作的特点在于可以简明地写出逆运算。例如
* a + x - x = a
* a ^ x ^ x = a

满足这种特性的算符使用前缀技术可以高效地求解区间查询：
$$
\begin{align}
&pre(n)\triangleq op_{k=1}^n a_k \\\\
&op(l, r) = pre(r)\space op^{-1}\space pre(l-1)
\end{align}
$$

但有些算符，例如 $\max, \min, or$ (同或操作)，它们不存在简明的逆运算符，从而无法用前缀技术解决区间问题 (例如，知道 $\max_{i\in [1, r]}a_i$ 和 $\max_{i\in [1, l-1]}a_i$ 对求解 $[l, r]$ 内的最大值没什么帮助)。但这些算符又具有加法、异或所不具备的特性——重复操作不会对结果带来影响：

* $\max(a, b) = \max(\max(\max(a, b), b), b)$
* $a | b = a | b | b | b$
  
ST表正适合用于解决这类算符的区间查询问题。ST表的本质是倍增思想 (由此可见倍增思想运用之广)。下面的讲解以最小值为例，但可以容易地扩展到其他算符上：

令 $ST(i, j)$ 表示区间 $[i, i + 2^j)$ 的最小值。容易发现对于一个数列来说，按照 $j$ 的顺序从小到大计算 ST 可以递推地高效求解：

$$
ST(i, j)=
\begin{cases}
a_i&, j = 0\\\\
\min(ST(i, j-1), ST(i + 2^{j-1}, j-1))&, j \geq 1
\end{cases}
$$

简单来说，长度为 $2^j$ 的区间的最小值可以用已经求好的两个长度为 $2^{j-1}$ 的区间的最小值求 $\min$ 得到。如果 $i + 2^{j-1}$ 超出了 $n$，那么可以不考虑后一半的贡献。

预处理好所有长度为2的次幂的区间的最小值，对于任意区间 $[l, r]$ 的最小值，我们可以用如下方式 $O(1)$ 地求解：令 $k$ 为最大的满足 $2^k\leq r-l+1$ 的整数，那么
$$
\min_{i=l}^r a_i = \min(ST(l, k), ST(r - 2^k + 1, k))
$$
它的思想是：$[l, l + 2^k)$ 和 $[r - 2^k + 1, r + 1)$ 这两个部分重叠的区间完整地覆盖了 $l[l, r]$，因此拿着它们的最小值求 min 即可得到全区间的最小值。在这里你可以看出 $\min$ 算符的“重复操作不影响结果”的特性对该做法的正确性保证至关重要。

最后一个简单的小问题是：如何求 $k$。从数学的角度，你可以使用数学库中的 $\log$ 函数，但更常见的做法是线性预处理所有可能长度对应的 $k$ 值：
```c++
K[1] = 0;
for (int i = 2; i <= n; i++) K[i] = K[i >> 1] + 1;
```
你可以用数学归纳法来证明该做法的正确性。

---

总结：
* ST 表可以在 $O(n\log n)$ 的预处理复杂度下，$O(1)$ 地求解任意区间的值。
* 算符需要保证具有“重复操作不影响结果”的特性。
* ST 表无法处理带修改的情况。

{{% alert tip %}}

**真的要具有“可重复贡献”性吗?**

如果你仔细学习了LCA章节的讲义，可能会想：如果把区间 $[l, r]$ 通过二进制拆分的方式拆成若干互不相交的，长度为2的次幂的区间的并，不就可以在 $O(\log n)$ 的时间内完成任意算符的区间查询了么？

非常好的想法！而且确实是正确的。只不过一般没有人会这么做——ST表不支持修改的特性是硬伤，它的“卖点”主要是 $O(1)$ 的效率。因此如果要花费 $O(\log n)$ 的代价才能完成区间求值，它的价值就十分有限了。我们会在合适的时机向大家展示支持修改的 $O(\log n)$ 地完成各种区间操作的技术。

{{% /alert %}}