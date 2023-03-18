---
title: "Lecture 02: Classical Probability and Geometric Probability"
linktitle: Lecture 02
type: docs
date: '2020-12-13T00:00:00Z'
draft: false

# Date published

# Is this an unpublished draft?

# Show this page in the Featured widget?
featured: false

math: true

menu:
    nju-probability:
        parent: Lectures
        weight: 2

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

## Classical Probability

古典概型的特点：

* $\Omega=\{e_1,e_2,\cdots,e_n\},n\in \mathbb{N}$。
* 对于任意 $1\leq i\leq n$，$P(\{e_i\})=\frac{1}{n}$ (即每个样本点等可能)。

在此基础上，若 $A=\{e_{i_1},\cdots e_{i_k}\}$ ，则 $P(A)=\frac{k}{n}$。

### Permutations and Combinations

* $n$ 个可分辨的球选 $r$ 个，可重复选，排列：$n^r$。

* $n$ 个可分辨的球选 $r$ 个，不可重复选，排列：$n^{\underline r}=\frac{n!}{(n-r)!}$。

* $n$ 个可分辨的球选 $r$ 个，不可重复选，组合：$\binom{n}{r}$。

* $n$ 个可分辨的球选 $r$ 个，可重复选，组合：$\binom{n+r-1}{r}$。

    > 考虑一个选择方案 $1\leq x_1\leq x_2\leq \cdots \leq x_r\leq n$，现设计另一个数列 $y$，满足 $y_1=x_1,y_2=x_2+1,y_3=x_3+2,\cdots,y_r=x_r+r-1$。那么 $y$ 严格单调增，且 $1\leq y_i\leq n+r-1$。$x$ 序列和 $y$ 序列一一对应，因此 $x$ 序列个数 = $y$ 序列个数 = $\binom{n+r-1}{r}$。

【例】求 $(a+b+c)^n$ 合并同类项的展开式中有多少项。

>  解：相当于求形如 $a^{n_1}b^{n_2}c^{n_3},n_1+n_2+n_3=n$ 的个数。可以将其理解为从 $3$ 个物品中选 $n$ 个，可重复选的组合方案数，因此答案为 $\binom{n+2}{n}=\frac{1}{2}(n+1)(n+2)$。

---

有关组合数的一些性质：

* $(1+x)^n=\sum_{k=0}^n\binom{n}{k}x^k$，令 $x=1$ ，可得 $\sum_{k=0}^n\binom{n}{k}=2^n$。
* $(1+x)^{a+b}=(1+x)^a(1+x)^b$，因此 $\binom{a+b}{n}=\sum_{k=0}^a\binom{a}{k}\binom{b}{n-k}$。若取 $a=b=n$，则可以得到 $\binom{2n}{n}=\sum_{k=0}^n\binom{n}{k}\binom{n}{n-k}=\sum_{k=0}^n\binom{n}{k}^2$。

## Geometrical Probability

$\fbox{Definition 2.1}$ **(几何概型)** 若 $\Omega$ 中的样本点与一个有界区域 $S$ 中的点一一对应，则事件 $A$ 对应于 $S$ 的一个子集 $D$。若 $A$ 的概率只和 $D$ 的测度有关，而与 $D$ 的形状，位置无关，那么 $P(A)=\frac{D的测度}{S 的测度}$。

【例】甲乙两人各在 1h 内随机一个时间点到达约会地点，先到的人最多等后到的人 15 分钟，求两人碰面的概率。

> 解：用数对 $(x,y)$ 表示甲乙两人到达的时间，则两人可以碰面当且即当 $|x-y|\leq 15$，画图：
>
> <img src="/img/NJU-22020170-02-01.png" style="zoom: 50%;" />
>
> 因此碰面概率为 $\frac{60^2-45^2}{60^2}=\frac{7}{16}$。

【例】 (蒲丰投针) 两平行线间距 $a$，向其投掷长度为 $l$ $(l<a)$ 的针，使针的中点在两平行线之间，求针与两条平行线中任意一条相交的概率。

> 解： $l<a$ 保证了针至多只会和一条线相交，根据对称性，我们只考虑针与下面的线相交的情况。
>
> 设针的中点与线的距离为 $x$，针所在的直线与线的夹角为 $\theta$，则针与线相交当且仅当 $x\leq \frac{l}{2}\sin\theta$。我们以 $x$ 和 $\theta$ 为坐标轴画出样本空间和相交事件：
> $$
> \begin{align}
> \Omega &= \{(\theta,x)|0<\theta<\pi,0\leq x\leq \frac{a}{2}\}\\\\
> A &= \{(\theta,x)|0\leq \theta \leq \pi, 0\leq x \leq \frac{l}{2}\sin \theta\}
> \end{align}
> $$
> $A$ 的图像是一个正弦函数，要计算面积，只需要计算积分：
> $$
> P(A)=\frac{1}{\pi\cdot \frac{a}{2}}\int_{0}^\pi\frac{l}{2}\sin \theta d\theta=\left.-\frac{l}{2}\cos\theta\right|_{0}^\pi=\frac{2l}{a\pi}
> $$
> 这个方法可以用于估算 $\pi$ 的值。通过大量重复试验用 $f_N(A)$ 来代替 $P(A)$ 后，$\pi=\frac{2l}{aP(A)}$。

【例】 (贝特朗奇论) 在一个半径为 1 的圆中等概率地取一根弦，弦长 $l>\sqrt 3$ 的概率是多少？

由于这里的“等概率”没有被严格地定义，因此可能有多种对等概率的解读，它们都是对的且会算出不同的结果：

1. 在圆周上固定一个点，然后另一个点在圆周上随机选取：$P(A)=\frac{1}{3}$。
2. 让一条直线从上往下均匀地扫一遍，发现只有中点距离圆心小于 $\frac{1}{2}$ 时弦长满足要求：$P(A)=\frac{1}{2}$。
3. 在圆内随机选取弦的中点，发现只有中点位于半径为 $\frac{1}{2}$ 的小圆内是弦长满足要求：$P(A)=\frac{1}{4}$。
4. ……