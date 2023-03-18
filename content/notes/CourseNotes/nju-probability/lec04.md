---
title: "Lecture 04: Independence and Bernoulli Experiments"
linktitle: Lecture 04
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
        weight: 4

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
--- 

## Independence

【例】$a$ 个黑球，$b$ 个白球，分别在有放回、无放回的情况下计算：

(1) 第一次摸到黑球 (A)，第二次摸到黑球的概率 (B)。

(2) 第二次摸到黑球的概率。

容易发现，有放回时，$P(B|A)=P(B)$ (放回后，第二次的实验条件与第一次的结果无关)，无放回时 $P(B|A)\neq P(B)$。

$\fbox{Definition 4.1}$ 称事件 $A$ 和 $B$ 互相独立，若 $P(AB)=P(A)P(B)$ (或者说，$P(B)=P(B|A)$)。

以下是一些推论：

* $\emptyset,\Omega$ 与任意事件独立。

* 若 $A,B$ 独立，则 $\overline{A},B$，$A,\overline{B}$ ，$\overline{A},\overline{B}$ 也独立。

    > Proof: 第一条：$P(\overline{A}B)=P(B-BA)=P(B)-P(BA)=P(B)-P(B)P(A)=P(B)(1-P(A))=P(B)P(\overline{A})$
    >
    > 容易推得第二，第三条也成立。$\blacksquare$

**注：事件独立和韦恩图上两个事件没有交集没有任何关系！**

$\fbox{Definition 4.2}$ (三个事件的独立性) $A,B,C$ 相互独立，若

* $A,B,C$ 两两互相独立。
* $P(ABC)=P(A)P(B)P(C)$。

注：这两条并不能互相推出，以下是例子：

* (1) 推不出 (2)：一个正四面体，一面红色 (A)，一面绿色 (B)，一面蓝色 (C)，一面三个颜色都有，讨论向下面的颜色：
    * $P(A)=P(B)=P(C)=2/4=1/2$，$P(AB)=1/4=P(A)P(B)$。
    * 然而 $P(ABC)=1/4\neq P(A)P(B)P(C)$。
* (2) 推不出 (1)：一个正八面体，1,2,3,4面有红色，1,2,3,5面有绿色，1,6,7,8面有蓝色。
    * $P(A)=P(B)=P(C)=4/8=1/2$，$P(ABC)=1/8=P(A)P(B)P(C)$。
    * 然而 $P(AB)=3/8\neq P(A)P(B)$。

类似地可以定义 $n$ 个事件的独立性：对于事件 $A_1,\cdots,A_n$，令 $I$ 为指标集，则它们独立当且仅当对于任意 $S\subseteq I$，有 $P(\bigcap_{\alpha\in S}A_\alpha)=\prod_{\alpha\in S}P(A_\alpha)$。

$\fbox{Theorem 4.3}$ 若事件 $A_1,\cdots, A_n$ 互相独立，那么考虑事件集的任意一个划分，每个组里的事件做任意运算的结果与别组的结果也互相独立。

$\fbox{Theorem 4.4}$ 若 $A_1,\cdots, A_n$ 相互独立，则 $P(\bigcup_{k=1}^nA_k)=1-\prod_{k=1}^nP(\overline{A_k})$。

证明：$P(\bigcup_{k=1}^nA_k)=1-P(\overline{\bigcup_{k=1}^nA_k})=1-P(\bigcap_{k=1}^n\overline{A_k})=1-\prod_{k=1}^nP(\overline{A_k})$。$\blacksquare$

## Bernoulli Experiments

$\fbox{Definition 4.5}$ 若一个独立重复试验满足

* 每个事件只有两个结果：$A$ 和 $\overline{A}$，$P(A)=p,P(\overline(A))=1-p$。
* 试验可重复，每两次试验之间互相独立。

则 $n$ 次上述实验称为 $n$ 重伯努利试验 (Bernoulli Experiment)。

$\fbox{Theorem 4.6}$ 伯努利试验中，记 $P_n(k)$ 为 $A$ 发生 $k$ 次的概率，则 $P_n(k)=\binom{n}{k}p^k(1-p)^{n-k}$。

【例】 (简单随机游走 simple random walk) 一个粒子从0出发在整数数轴上运动，每次向右移动的概率为 $p$，求跳 $n$ 次后位于 $k$ 的概率。

> 解：以下只讨论 $k\geq 0$ 的情况：
> $$
> P(n,k)=
> \begin{cases}
> 0&, k>n\\\\
> 0&, n与k奇偶性不同\\\\
> \binom{n}{(n+k)/2}p^{(n+k)/2}(1-p)^{(n-k)/2}&，otherwise
> \end{cases}
> $$

$\fbox{Theorem 4.7}$ (泊松定理, Poisson) 若 $np_n=\lambda$，则对于固定的 $k$，
$$
\lim_{n\rightarrow \infty} \binom{n}{k}p_n^k(1-p_n)^{n-k}=\frac{\lambda^k}{k!}e^{-\lambda}
$$
证明：
$$
\begin{align}
\lim_{n\rightarrow \infty}\binom{n}{k}p_n^k(1-p_n)^{n-k}&=\lim_{n\rightarrow \infty}\frac{n(n-1)\cdots (n-k+1)}{k!}(\frac{\lambda}{n})^k(1-\frac{\lambda}{n})^{n-k}\\\\
&=\lim_{n\rightarrow \infty}\frac{\lambda^k}{k!}\cdot 1(1-\frac{1}{n})(1-\frac{2}{n})\cdots (1-\frac{k-1}{n})(1-\frac{\lambda}{n})^n(1-\frac{\lambda}{n})^{-k}\\\\
&=\lim_{n\rightarrow \infty}\frac{\lambda^k}{k!}(1-\frac{\lambda}{n})^n\\\\
&=\frac{\lambda^k}{k!}\left[\lim_{n\rightarrow \infty}(1+\frac{-\lambda}{n})^{\frac{n}{-\lambda}}\right]^{-\lambda}\\\\
&=\frac{\lambda^k}{k!}e^{-\lambda}
\end{align}
$$