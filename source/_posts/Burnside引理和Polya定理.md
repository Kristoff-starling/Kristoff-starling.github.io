---
layout: post
date: 2021-11-12
title: "Burnside引理和Polya定理"
categories:
tags: [数学笔记, 群论]
mathjax: true
---

## Burnside引理

### 描述

设 $A$ 和 $B$ 为有限集，$X$ 是一些从 $A$ 到 $B$ 的映射的集合。$G$ 是 $A$ 上的置换群，且 $X$ 中的映射在 $G$ 的作用下封闭，那么
$$
|X/G|=\frac{1}{|G|}\sum_{g\in G}|X^g|
$$
其中 $X/G$ 表示 $X$ 在 $G$ 的作用下所有等价类的集合（对于 $x,y\in X$，若存在 $g\in G$, $g(x)=y$，则 $x,y$ 在同一个等价类中）。$X^g=\\{x\in X:g(x)=x\\}$。

<!-- more -->

### 证明

首先给出**轨道稳定子定理**：$X$ 和 $G$ 的定义同上，对于任意 $x\in X$，称 $G^x=\\{g\in G:g(x)=x\\}$ 为 $x$ 的**稳定子**，$G(x)=\\{g(x):g\in G\\}$ 为 $x$ 的**轨道**，则
$$
|G|=|G^x||G(x)|
$$

> 证明：首先证明 $G^x$ 是 $G$ 的一个子群：恒等置换 $I$ 满足 $I(x)=x$，所以 $I\in G^x,G^x\neq \emptyset$ 。对于任意 $g\in G^x,h\in G^x$，考虑 $h$ 在 $G$ 中的逆元 $h^{-1}$，因为 $h^{-1}(x)=h^{-1}(h(x))$$=(h^{-1}\circ h)(x)=I(x)=x$，所以 $h^{-1}\in G^x$，于是 $gh^{-1}(x)=g(h^{-1}(x))=g(x)=x$，所以 $gh^{-1}\in G^x$。所以 $G^x$ 是 $G$ 的子群。
>
> 根据拉格朗日定理，$|G^x||G:G^x|=|G|$，其中 $G:G^x$ 表示 $G^x$ 的所有左陪集构成的集合。所以现在只要证明 $|G(x)|=|G:G^x|$。考虑映射 $\varphi: G(x)\rightarrow G:G^x$，$\varphi(g(x))=gG^x$，下证 $\varphi$ 是双射：
>
> * 如果 $g(x),f(x)\in G(x)$ 满足 $gG^x=fG^x$，两边左乘 $f^{-1}$ 有 $(f^{-1}g)G^x=G^x$，所以有 $f^{-1}g\in G^x$，$(f^{-1}\circ g)(x)=x$。两边左乘 $f$ 即有 $g(x)=f(x)$，所以 $\varphi$ 是单射。
> * 对于 $G:G^x$ 中的任意一个左陪集 $gG^x$，存在 $g(x)\in G(x)$，$\varphi(g(x))=gG^x$，所以 $\varphi$ 是满射。
>
> 结合以上两点，$\varphi$ 是双射，从而有 $|G(x)|=|G:G^x|$。
>
> 综上，轨道稳定子定理得证。$\blacktriangle$ 

再考虑 Burnside 引理的证明，我们有

$$
\begin{align}
\sum_{g\in G}|X^g| &= |\\{(g,x)\in G\times X: g(x)=x\\}|\\\\
&= \sum_{x\in X}|G^x|\\\\
&= \sum_{x\in X}\frac{|G|}{|G(x)|}\quad (轨道稳定子定理)\\\\
&= |G|\sum_{Y\in X/G}\sum_{x\in Y}\frac{1}{|G(x)|}\\\\
&= |G|\sum_{Y\in X/G}\sum_{x\in Y}\frac{1}{|Y|}\quad (\*)\\\\
&= |G||X/G|
\end{align}
$$
注： $(*)$ 一步的道理来自映射集合 $X$ 在 $G$ 的作用下的等价类的含义。事实上对于任意 $x\in G$，$x$ 的轨道就是 $x$ 所在的等价类。

## Polya定理

### 描述

在 Burnside 引理的基本条件的基础上，若 $X$ 是全部从 $A$ 到 $B$ 映射的集合（那么显然有 $X$ 在 $G$ 的作用下封闭），那么
$$
|X/G|=\frac{1}{|G|}\sum_{g\in G}|B|^{c(g)}
$$
其中 $c(g)$ 表示置换 $g$ 中的轮换（循环子置换）个数。

### 证明

事实上 Polya 定理是在 Burnside 引理的基础上给出了一种高效计算 $|X^g|$ 的方法。 当 $X$ 是全部从 $A$ 到 $B$ 的映射的集合时，映射 $\phi$ 在 $g$ 的作用下是不动映射的充要条件是对于 $g$ 中的每个轮换，轮换中元素对应的 $\phi$ 值相同。$\phi$ 值有 $|B|$ 种选择，从而根据乘法原理，不动映射的个数是 $|B|^{c(g)}$。