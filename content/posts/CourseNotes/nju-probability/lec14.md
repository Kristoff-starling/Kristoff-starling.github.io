---
title: "Lecture 14: Variance"
linktitle: Lecture 14
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
        weight: 14

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

我们希望衡量一个随机变量每个取值和平均值的差异，并且希望这个值是正数。因此我们引入方差的概念。

$\fbox{Definition 14.1}$ 设 $X$ 是一个随机变量。若 $E[|X|^2]$ 存在，则称
$$
D(X)\triangleq E[(X-E[x])^2]
$$
为 $X$ 的方差，同时称 $\sigma(X)\triangleq \sqrt{D(X)}$ 为 $X$ 的均方差/标准差。

注：(1) $(X-E[X])^2$ 的数学性质好于 $|X-E[X]|$，比如可导性，以及代数运算时的方便性。

(2) $E[X^2]<+\infty$ 可以推出 $E[X]<+\infty$ (反过来不成立)。

(3) 对于离散型随机变量，设 $P(X=x_i)=p_i$，则
$$
D(X)=\sum_{k=1}^\infty(x_k-E[X])^2p_k
$$
对于连续型随机变量，设密度函数为 $p(x)$，则
$$
D(X)=\int_{-\infty}^{+\infty}(x-E[X])^2p(x)dx
$$
(4) 方差公式还有另外一种形式：
$$
\begin{align}
D(X)&=E[(X-E[X])^2]\\\\
&=E[X^2-2E[X]X+E[X]^2]\\\\
&=E[X^2]-E[2E[X]X]+E[E[X]^2]\\\\
&=E[X^2]-2E[X]^2+E[X]^2\\\\
&=E[X^2]-E[X]^2
\end{align}
$$
从这个公式中我们可以看出：$D(X)\leq E[X^2]$。

【例】考虑 $X\sim B(n,p)$ 的方差。$X$ 的分布律为 $P(X=k)=\binom{n}{k}p^k(1-p)^{n-k}$。我们已知 $E[X]=np$，因此只需要求 $E[X^2]$。
$$
\begin{align}
E[X^2]&=\sum_{k=1}^nk^2\frac{n!}{k!(n-k)!}p^k(1-p)^{n-k}\\\\
&=\sum_{k=1}^nk\frac{n!}{(k-1)!(n-k)!}p^k(1-p)^{n-k}\\\\
&=\sum_{k=1}^n(k-1)\frac{n!}{(k-1)!(n-k!)}p^k(1-p)^{n-k}+\sum_{k=1}^n\frac{n!}{(k-1)!(n-k)!}p^k(1-p)^{n-k}\\\\
&=n(n-1)p^2\sum_{k=2}^n\frac{(n-2)!}{(k-2)!(n-k)!}p^{k-2}(1-p)^{n-k}+np\sum_{k=1}^n\frac{(n-1)!}{(k-1)!(n-k)!}p^{k-1}(1-p)^{n-k}\\\\
&=n(n-1)p^2[1+(1-p)]^{n-2}+np[1+(1-p)]^{n-1}\\\\
&=n(n-1)p^2+np
\end{align}
$$
因此 $D(X)=E[X^2]-E[X]^2=np-np^2=np(1-p)$。

【例】考虑 $X\sim p(\lambda)$ 的泊松分布的方差。$X$ 的分布律为 $P(X=k)=\frac{\lambda^k}{k!}e^{-\lambda}$，我们已知 $E[X]=\lambda$，因此只需要求 $E[X^2]$。
$$
\begin{align}
E[X^2]&=\sum_{k=1}^\infty k^2\frac{\lambda^k}{k!}e^{-\lambda}=\sum_{k=1}^\infty k\frac{\lambda^k}{(k-1)!}e^{-\lambda}\\\\
&=\sum_{k=1}^\infty(k-1)\frac{\lambda^k}{(k-1)!}e^{-\lambda}+\sum_{k=1}^\infty\frac{\lambda^k}{(k-1)!}e^{-\lambda}\\\\
&=\lambda^2\sum_{k=2}^\infty\frac{\lambda^{k-2}}{(k-2)!}e^{-\lambda}+\lambda\sum_{k=1}^\infty\frac{\lambda^{k-1}}{(k-1)!}e^{-\lambda}\\\\
&=\lambda^2+\lambda
\end{align}
$$
因此 $D(X)=E[X^2]-E[X]^2=\lambda$。

【例】 考虑指数分布 $X\sim E(\lambda)$ 的方差。$X$ 的密度函数为 $p(x)=\begin{cases}\lambda e^{-\lambda x}&,x>0\\\\0&,x\leq 0\end{cases}$。我们已知 $E[X]=\frac{1}{\lambda}$，因此只需要求 $E[X^2]$。
$$
\begin{align}
E[X^2]&=\int_{-\infty}^{+\infty}x^2p(x)dx=\int_0^{+\infty}x^2\lambda e^{-\lambda x}dx=-\int_0^{+\infty}x^2e^{-\lambda x}d(-\lambda x)\\\\
&=-\int_0^{+\infty}x^2d(e^{-\lambda x})=\left.x^2e^{-\lambda x}\right|_0^{+\infty}+2\int_0^{+\infty}xe^{-\lambda x}dx\\\\
&=\frac{2}{\lambda}\int_0^{+\infty}x\lambda e^{-\lambda x}dx\\\\
&=\frac{2}{\lambda}E[X]=\frac{2}{\lambda^2}
\end{align}
$$
因此 $D(X)=E[X^2]-E[X]^2=\frac{1}{\lambda^2}$。

【例】考虑正态分布 $X\sim N(\mu, \sigma^2)$ 的方差。$X$ 的密度函数为 $p(x)=\frac{1}{\sqrt {2\pi }\sigma}e^{-\frac{(x-\mu)^2}{2\sigma^2}}$。我们容易发现这里直接方差的原始定义式计算会更加方便：
$$
\begin{align}
D(X)&=E[(X-E[X])^2]=\frac{1}{\sqrt{2\pi}\sigma}\int_{-\infty}^{+\infty}(x-\mu)^2e^{-\frac{(x-\mu)^2}{2\sigma^2}}dx\\\\
&\overset{y=\frac{x-\mu}{\sigma}}{=}\frac{\sigma^2}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}y^2e^{-\frac{y^2}{2}}dy\\\\
&=-\frac{\sigma^2}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}yd\left(e^{-\frac{y^2}{2}}\right)\\\\
&=\left.-\frac{\sigma^2}{\sqrt{2\pi}}ye^{-\frac{y^2}{2}}\right|\_{-\infty}^{+\infty}+\sigma^2\cdot \frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}e^{-\frac{y^2}{2}}dy\\\\
&=0+\sigma^2\cdot (标准正态分布在 \mathbb R 上的密度函数积分)\\\\
&=\sigma^2
\end{align}
$$
方差的性质：

* 若 $X\equiv a$，则 $D(X)=E[(X-a)^2]=0$ 。

* 若 $D(X)$ 存在，对于任意 $a,b\in \mathbb R$，$D(aX+b)=a^2D(X)$。

    > 证明：
    > $$
    > \begin{align}
    > D(aX+b)&=E[(aX+b-E[aX+b])^2]\\\\
    > &=E[(aX+b-aE[X]-b)^2]\\\\
    > &=E[(a(X-E[X]))^2]\\\\
    > &=a^2D(X)
    > \end{align}
    > $$

    直观来看这个公式也容易理解：全局加上 $b$ 不影响对期望的偏离；全局乘 $a$ 在 L2 偏差下最终反映为 $a^2$。

    在该性质的支撑下，对于随机变量 $X$，若 $D(X)$ 存在，考虑 $Z=\frac{X-E[X]}{\sqrt{D(X)}}$，则 $E[Z]=0$，$D(Z)=\left(\frac{1}{\sqrt{D(X)}}\right)^2D(X)=1$，因此称 $Z$ 为 $X$ 的标准化。(标准正态分布和正态分布之间的转化是该公式的一个特例)。

* 设 $X,Y$ 为随机变量，则 $D(X\pm Y)=D(X)+D(Y)\pm 2E[(X-E[X])(Y-E(Y))]$。

    > 证明：这里只证 + 的情况，- 的情况同理。
    > $$
    > \begin{align}
    > D(X+Y)&=E[(X+Y-E[X+Y])^2]=E[((X-E[X])+(Y-E[Y]))^2]\\\\
    > &=E[(X-E[X])^2]+E[(Y-E[Y])^2]+2E[(X-E[X])(Y-E[Y])]\\\\
    > &=D(X)+D(Y)+2E[(X-E[X])(Y-E[Y])]
    > \end{align}
    > $$

    特别地，若 $X,Y$ 独立，则 $X-E[X], Y-E[Y]$ 也独立，则
    $$
    E[(X-E[X])(Y-E[Y])]=E[X-E[X]]E[Y-E[Y]]=0
    $$
    从而 $D(X\pm Y)=D(X)+D(Y)$ (注意符号！)。

    上述结论可以推广到 $n$ 个随机变量的情形：
    $$
    D\left(\sum_{i=1}^nx_i\right)=\sum_{i=1}^nD(x_i)-\sum_{1\leq i<j\leq n}E[(x_i-E[x_i])(x_j-E[x_j])]
    $$
    若这些变量两两独立 (注意，该条件比 $n$ 个变量的独立性弱！)，则
    $$
    D\left(\sum_{i=1}^nx_i\right)=\sum_{i=1}^nD(x_i)
    $$

* 对于任意 $c\in \mathbb R$，$D(X)\leq E[(X-c)^2]$。

    > 证明：
    > $$
    > \begin{align}
    > D(X)&=E[(X-E[X])^2]=E[(X-c+c-E[X])^2]\\\\
    > &=E[(X-c)^2]+2E[(X-c)(c-E[X])]+E[(c-E[X])^2]\\\\
    > &=E[(X-c)^2]+2(c-E[X])(E[X]-c)+(c-E[X])^2\\\\
    > &=E[(X-c)^2]-(c-E[X])^2\\\\
    > &\leq E[(X-c)^2]
    > \end{align}
    > $$

利用期望的可拆分性 (性质 3) 有时能大幅简化计算过程，这里以二项分布 $X\sim B(n,p)$ 为例。令 $x_i$ 为描述第 $i$ 次实验是否成功的示性随机变量，显然 $x_1,\cdots, x_n$ 两两独立。那么
$$
\begin{align}
D(X)&=D\left(\sum_{i=1}^nx_i\right)=\sum_{i=1}^nD(x_i)=nD(x_1)\\\\
&=nE[(x_1-E[x_1])^2]=n((1-p)^2p+(0-p)^2(1-p))\\\\
&=np(1-p)
\end{align}
$$