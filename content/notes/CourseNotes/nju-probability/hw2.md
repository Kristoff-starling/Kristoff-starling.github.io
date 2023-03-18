---
title: "Homework 2"
linktitle: Homework 2
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
        parent: Homework
        weight: 2

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

## Problem 2.1

解：
$$
\begin{align}
P(X=1)&=\frac{\binom{4}{3}\cdot 6}{4^3}=\frac{3}{8}\\\\
P(X=2)&=\frac{\binom{3}{2}\binom{4}{2}\cdot 2}{4^3}=\frac{9}{16}\\\\
P(X=3)&=\frac{4}{4^3}=\frac{1}{16}
\end{align}
$$
综上，$X$ 的分布列为

| $X$    | $1$           | $2$            | $3$            |
| :------: | :-------------: | :--------------: | :--------------: |
| $P(X)$ | $\frac{3}{8}$ | $\frac{9}{16}$ | $\frac{1}{16}$ |

## Problem 2.4

解：各个常数的计算过程如下：

* $a=\frac{3}{4}-P(X=-1)=\frac{1}{2}$。
* $b=1-P(X=-1)-P(X=0)=\frac{1}{4}$。

* $c=P(X<-1)=0$。
* $d=P(X<0)=P(X=-1)=\frac{1}{4}$。
* $e=P(X=-1)+P(X=0)+P(X=1)=1$。

## Problem 2.8

解：(1)
$$
\begin{align}
P(X\leq 3)&=e^{-\lambda}\sum_{k=0}^3\frac{\lambda^k}{k!}\approx 0.7576\\\\
P(转港)&=P(X>3)=1-P(X\leq 3)=0.2424
\end{align}
$$
(2) 考察函数 $f(x)=\frac{2.5^x}{x!},x\in \mathbb N$，显然当 $x=2$ 时 $f(x)$ 取得最大值。所以最大可能到达港口的油船数为 2，概率为 $\frac{2.5^2}{2}e^{-2.5}\approx 0.2565$。

(3) 注意到
$$
\begin{align}
P(X\leq 4)&=e^{-\lambda}\sum_{k=0}^4\frac{\lambda^k}{k!}=0.8912<0.9\\\\
P(X\leq 5)&=e^{-\lambda}\sum_{k=0}^5\frac{\lambda^k}{k!}=0.9553\geq 0.9
\end{align}
$$
所以服务能力提高到 5 只油船才能使得到达游船以 $90\%$ 的概率得到服务。

## Problem 2.12

解：(1)
$$
\int_{-\infty}^{+\infty}p(x)dx=\int_0^1p(x)dx=A\int_0^1x^3dx=\left.\frac{A}{4}x^4\right|_0^1=\frac{A}{4}=1
$$
解得 $A=4$。

(2)
$$
F(x)=P(X\leq x)=\int_{-\infty}^xp(u)du=
\begin{cases}
0&,x\leq 1\\\\
x^4&,0<x\leq 1\\\\
1&,x>1
\end{cases}
$$
(3) 令 $F(x)=0.5$，解得 $x=(\frac{1}{2})^{\frac{1}{4}}$，所以 $B=\left(\frac{1}{2}\right)^{\frac{1}{4}}$。

## Problem 2.16

解：令 $Z=\frac{X-\mu}{4},W=\frac{Y-\mu}{5}$，则 $Z\sim N(0,1),W\sim N(0,1)$。
$$
\begin{align}
p_1&=P(X\leq \mu-4)=P(4Z+\mu\leq \mu-4)=P(Z\leq -1)=P(Z\geq 1)\\\\
p_2&=P(Y\geq \mu+5)=P(5W+\mu\geq \mu+5)=P(W\geq 1)
\end{align}
$$
又 $Z,W$ 分布函数相同，所以 $p_1=p_2$。

## Problem 2.19

解：(1) $Y=2X$ 的分布律如下：

| $Y$  | $-4$          | $-1$          | $0$           | $1$           | $8$           |
| :----: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: |
| $P$  | $\frac{1}{8}$ | $\frac{1}{4}$ | $\frac{1}{8}$ | $\frac{1}{6}$ | $\frac{1}{3}$ |

(2) $Y=X^2$ 的分布律如下：

| $Y$  | $0$           | $\frac{1}{4}$  | $4$           | $16$          |
| :----: | :-------------: | :--------------: | :-------------: | :-------------: |
| $P$  | $\frac{1}{8}$ | $\frac{5}{12}$ | $\frac{1}{8}$ | $\frac{1}{3}$ |

(3) $Y=\sin \left(\frac{\pi}{2}X\right)$ 的分布律如下：

| $Y$  | $-\frac{\sqrt 2}{2}$ | $0$            | $\frac{\sqrt 2}{2}$ |
| :----: | :--------------------: | :--------------: | :-------------------: |
| $P$  | $\frac{1}{4}$        | $\frac{7}{12}$ | $\frac{1}{6}$       |

## Problem 2.26

证明：$X$ 服从参数为 2 的指数分布，即
$$
p_X(x)=\begin{cases}
2e^{-2x}&,x\geq 0\\\\
0&,x<0
\end{cases}
$$
$Y=g(X)=1-e^{-2X}$，$g'(x)=-e^{-2x}\cdot (-2)=2e^{-2x}>0$，所以 $g(x)$ 严格单调递增且处处可导。因此
$$
\begin{align}
p_Y(y)&=p_X(g^{-1}(y))\cdot \left|g^{-1}(y)\right|\\\\
&=\frac{1}{2(1-y)}p_X(-\frac{1}{2}\ln(1-y))\\\\
&=
\begin{cases}
1&,0\leq x\leq 1\\\\
0&, otherwise
\end{cases}
\end{align}
$$
 可以看到在 $[0,1]$ 中，$p_Y(y)=1=\frac{1}{1-0}$，所以 $Y$ 在 $[0,1]$ 上服从均匀分布。$\blacksquare$

