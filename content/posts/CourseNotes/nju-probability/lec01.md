---
title: "Lecture 01: Probability Space"
linktitle: Lecture 01
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
        weight: 1

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

我们可以通过随机试验来研究随机现象。随机试验应当具有以下特点：

* 相同条件下可重复；
* 试验结果不唯一，试验前未知，但所有可能的结果已知；

概率空间是一个三元组 $(\Omega, \mathscr F, P)$，其中 $\Omega$ 为样本空间，$\mathscr F$ 为可测事件集，$P$ 为概率测度。

## Sample Space

$\fbox{Definition 1.1}$ 随机试验的所有可能结果的集合称为**样本空间 (sample space)**，记作 $\Omega$。每个随机试验的可能结果称为**样本点/基本事件 (sample point)**，记作 $e$ 或 $\omega$，$e\in \Omega$。

## Measurable Event Set 

$\fbox{Definition 1.2.1}$ 称 $\mathscr{F}\subseteq Pot(\Omega)$ 是集合 $\Omega$ 上的一个 **$\sigma$-域 ($\sigma$-field)**，当且仅当

* $\Omega \in \mathscr{F}$；
* 若 $A\in \mathscr F$，则 $A^C\in \mathscr F$；
* 若 $A_1,\cdots A_n\cdots \in \mathscr F$，则 $\bigcup_{i=1}^\infty A_i\in \mathscr F$ (即可数个集合的并也是 $\mathscr F$ 的元素)。

$\fbox{Definition 1.2.2}$ **可测事件集** $\mathscr F$ 是 $\Omega$ 上的一个 $\sigma$-域，若 $A\subseteq \Omega$，$A\in \mathscr F$，则称 $A$ 是一个**事件 (event)**。

## Probability Metric

$\fbox{Definition 1.3}$ 集合函数 $P:\mathscr F\rightarrow [0,1]$ 是 $(\Omega, \mathscr F)$ 上的一个**概率测度 (probability metric)**，当且仅当

* $P(\Omega)=1$；
* 对于任意 $A\in \mathscr F$，$P(A)\geq 0$；
* 满足可列可加性：对于<u>互不相容</u>的事件 $A_1,\cdots,A_n\cdots\in \mathscr F$，$P(\bigcup_{n=1}^\infty A_n)=\sum_{n=1}^\infty P(A_n)$。

以下是一系列有用的推论：

1. $P(\emptyset)=0$。

    >  Proof: 取 $A_1=\Omega, A_2=A_3=\cdots =\emptyset$，那么
    >  $$
    >  P(\Omega)=P(\bigcup_{n=1}^\infty A_n)=\sum_{n=1}^\infty P(A_n)=P(\Omega)+\sum_{n=2}^\infty P(\emptyset)
    >  $$
    >  所以 $P(\emptyset)=0$。$\blacksquare$

2. 可列可加性可以推出有限可加性，即对于 $n\in \mathbb{N}$ ， $P(\bigcup_{k=1}^nA_k)=\sum_{k=1}^nP(A_k)$。

    > Proof: 取 $A_{n+1}=A_{n+2}=\cdots =\emptyset$，则有
    > $$
    > \begin{align}
    > P(\bigcup_{k=1}^nA_k)&=P(\bigcup_{k=1}^\infty A_k)=\sum_{k=1}^\infty P(A_k)\\\\
    &=\sum_{k=1}^nP(A_k)+\sum_{k=n+1}^\infty P(\emptyset)=\sum_{k=1}^nP(A_k)
    > \end{align}
    > $$
    > 有限可加性得证。$\blacksquare$

3. $P(A-B)=P(A)-P(A\cap B)$。

    > Proof: 由 $A=(A-B)\cup (A\cap B)$ 和结论 2 易得。$\blacksquare$

4. $P(A\cup B)=P(A)+P(B)-P(A\cap B)$。更一般地，
    $$
    P(\bigcup_{k=1}^nA_k)=\sum_{i=1}^n\left((-1)^{i+1}\sum_{1\leq j_1<j_2<\cdots<j_i\leq n}P(\bigcap_{k=1}^iA_{j_i})\right)
    $$
    (容斥原理的表达式)。

