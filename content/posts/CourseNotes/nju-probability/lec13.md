---
title: "Lecture 13: Mathematical Expectation"
linktitle: Lecture 13
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
        weight: 13

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

## Expectation of Discrete Random Variables

离散型随机变量的数学期望类似于“加权平均数”，但我们要将频率换成严格的概率。

$\fbox{Definition 13.1}$ 设离散型随机变量 $X$ 的分布律为 $P(X=x_i)=p_i,i=1,2,\cdots$。若级数 $\sum_{i=1}^\infty |x_i|p_i$ 收敛，则称
$$
E[x]\triangleq \sum_{i=1}^\infty x_ip_i
$$
为 $X$ 的数学期望，简称均值或期望。

注：(1) 分布唯一决定了期望。

(2) 在定义中，我们要求级数 $\sum_{i=1}^\infty x_ip_i$ 绝对收敛，这是为了保证重排 $x_ip_i$ 的顺序不影响期望的值 (采样顺序不应影响均值)。

(3) 当 $\sum_{i=1}^\infty |x_i|p_i$ 不收敛时，称 $X$ 的期望不存在。

【例】考虑泊松分布 $P(\lambda)$ 的期望。泊松分布的分布律为
$$
P(X=k)=\frac{\lambda^k}{k!}e^{-\lambda}\qquad k=0,1,\cdots
$$
因此
$$
\begin{align}
E[x]&=\sum_{k=1}^\infty kP(X=k)=\sum_{k=1}^\infty k\frac{\lambda^k}{k!}e^{-\lambda}=e^{-\lambda}\sum_{k=1}^\infty \frac{\lambda^k}{(k-1)!}\\\\
&=e^{-\lambda}\lambda\cdot \sum_{n=0}^{\infty} \frac{\lambda^n}{n!}=e^{-\lambda}\cdot \lambda\cdot e^{\lambda}\\\\
&=\lambda
\end{align}
$$
【例】考虑二项分布 $X\sim B(n,p)$ 的期望。二项分布的分布律为
$$
P(X=k)=\binom{n}{k}p^k(1-p)^{n-k}\qquad k=0,1,\cdots, n
$$
因此
$$
\begin{align}
E[x]&=\sum_{k=1}^\infty kP(X=k)=\sum_{k=1}^n k\binom{n}{k}p^k(1-p)^{n-k}\\\\
&=\sum_{k=1}^n \frac{n!}{(k-1)!(n-k)!}p^k(1-p)^{n-k}\\\\
&=np\sum_{k=1}^n\frac{(n-1)!}{(k-1)!(n-k)!}p^{k-1}(1-p)^{n-k}\\\\
&=np\sum_{k'=0}^{n-1}\binom{n-1}{k'}p^{k'}(1-p)^{n-1-k'}\\\\
&=np\cdot [p+(1-p)]^{n-1}\\\\
&=np
\end{align}
$$
【例】考虑几何分布 $X\sim g(p)$ 的期望。几何分布的分布律为
$$
P(X=k)=(1-p)^{k-1}p\qquad k=1,2,\cdots
$$
因此
$$
\begin{align}
E[x]&=\sum_{k=1}^\infty kP(X=k)=\sum_{k=1}^\infty k(1-p)^{k-1}p=p\sum_{k=1}^\infty k(1-p)^{k-1}
\end{align}
$$
令
$$
F(p)=\sum_{k=1}^\infty (1-p)^k=\frac{1-p}{p}
$$
(注：$1-p\in [0,1]$ 在收敛半径内，因此等式成立，后续的求导和求和可以交换。)

则
$$
\frac{d}{dp}F(p)=-\sum_{k=1}^\infty k(1-p)^{k-1}
$$
所以
$$
E[x]=-pF'(p)=-p\cdot \left(\frac{1-p}{p}\right)'=-p\cdot -\frac{1}{p^2}=\frac{1}{p}	
$$
【例】 (圣彼得堡悖论) 连续扔一枚均匀硬币，当出现反面时停止。若此前出现的正面次数为 $k$ 次，则收益为 $2^k$，求该游戏收益的期望。

> 解：设收益为 $X$，则 $P(X=2^k)=\frac{1}{2^{k+1}},k=0,1,\cdots$
> $$
> E[x]=\sum_{k=0}^\infty x_kP(X=x_k)=\sum_{k=0}^\infty 2^k\frac{1}{2^{k+1}}=\sum_{k=0}^\infty \frac{1}{2}=+\infty
> $$
> 因此期望不存在。

这是一个期望不存在的例子。有其他角度可以研究如何定价可以使该游戏公平。一个合理的定价是支付 $n\log_2 n$ 玩 $n$ 次。

## Expectation of Continuous Random Variables

$\fbox{Definition 13.2}$ 设连续型随机变量 $X$ 的密度为 $p(x)$，若积分 $\int_{-\infty}^{+\infty}|x|p(x)dx<+\infty$，则称
$$
E[x]\triangleq \int_{-\infty}^{+\infty}xp(x)dx
$$
为 $X$ 的期望。

注：若对 $\mathbb R$ 取很密的划分
$$
\begin{align}
&0=x_0<x_1<\cdots <x_n<\cdots<+\infty\\\\
&0=y_0>y_1>\cdots >y_n>\cdots>-\infty
\end{align}
$$
把 $X$ 近似看成”随机变量“ $\tilde{X}$，满足
$$
\begin{align}
\tilde P(\tilde X=x_k)&=p(x_k)(x_{k+1}-x_k),k\geq 0\\\\
\tilde P(\tilde X=y_k)&=p(y_k)(y_{k-1}-y_k),k\geq 1
\end{align}
$$
则
$$
\tilde E[\tilde X]=\sum_{k=0}^\infty x_kp(x_k)(x_{k+1}-x_k)+\sum_{k=1}^\infty y_kp(y_k)(y_{k-1}-y_k)
$$
为 $E[x]$ 的渐进和式。

【例】考虑指数分布 $X\sim E(\lambda)$ 的期望。指数分布的密度为
$$
p(x)=\begin{cases}\lambda e^{-\lambda x}&,x\geq 0\\\\0&,x<0\end{cases}
$$
因此

$$
\begin{align}
E[x]&=\int_{-\infty}^{+\infty}xp(x)dx=\int_0^{+\infty}x\cdot \lambda e^{-\lambda x}dx=-\int_0^{+\infty}xe^{-\lambda x}d(-\lambda x)=-\int_{0}^{+\infty}xd(e^{-\lambda x})\\\\
&=\left .-xe^{-\lambda x}\right|_{0}^{\infty}+\int_0^{+\infty}e^{-\lambda x}dx=-\frac{1}{\lambda}\int_0^{+\infty}e^{-\lambda x}d(-\lambda x)=\left.-\frac{e^{-\lambda x}}{\lambda}\right|_0^{+\infty}\\\\
&=\frac{1}{\lambda}
\end{align}
$$

【例】考虑正态分布 $X\sim N(\mu,\sigma^2)$ 的期望。正态分布的密度为
$$
p(x)-\frac{1}{\sqrt{2\pi}\sigma}e^{-\frac{(x-\mu)^2}{2\sigma^2}}
$$
因此
$$
\begin{align}
E[x]&=\int_{-\infty}^{+\infty}xp(x)dx=\int_{-\infty}^{+\infty}x\cdot \frac{1}{\sqrt{2\pi}\sigma}e^{-\frac{(x-\mu)^2}{2\sigma^2}}dx\\\\
&\overset{y=x-\mu}{=}\int_{-\infty}^{+\infty}(y+\mu)\frac{1}{\sqrt{2\pi}\sigma}e^{-\frac{y^2}{2\sigma^2}}dy\\\\
&=\frac{1}{\sqrt{2\pi}\sigma}\int_{-\infty}^{+\infty}ye^{-\frac{y^2}{2\sigma^2}}dy+\mu\cdot \frac{1}{\sqrt{2\pi}\sigma}\int_{-\infty}^{+\infty}e^{-\frac{y^2}{2\sigma^2}}dy
\end{align}
$$
第一项是奇函数在对称区域上的积分，为 0。第二项除了 $\mu$ 后面的部分正好是正态分布全区域上的 $p(x$) 的积分，为 1，因此 $E[x]=\mu$。

【例】 连续型随机变量的期望也可能不存在。考虑柯西分布 $X$，$p(x)=\frac{1}{\pi(1+x^2)}$，求其期望。

> 解：
> $$
> \begin{align}
> \int_{-\infty}^{+\infty}|x|p(x)dx&=\int_{-\infty}^{+\infty}\frac{|x|dx}{\pi(1+x^2)}=2\int_0^{+\infty}\frac{xdx}{\pi(1+x^2)}\\\\
> &=\frac{1}{\pi}\int_0^{+\infty}\frac{d(1+x^2)}{1+x^2}=\left.\frac{1}{\pi}\ln (1+x^2)\right|_{0}^{+\infty}=+\infty
> \end{align}
> $$
> 因此期望不存在。

注：微妙的一点是：柯西分布的密度函数是关于 $y$ 轴对称的，但这并不能说明期望存在且等于 0。

## Expectation of Random Variable Functions

设 $X$ 的分布已知，$Y=g(X)$，可通过下面的定理求 $E[Y]$ 而不必先求出 $Y$ 的分布再求期望：

$\fbox{Theorem 13.3}$ 若 $X$ 为离散随机变量，分布律为 $P(X=x_k)=p_k,k=1,2,\cdots$，若 $\sum_{k=1}^\infty |g(x_k)|p_k<+\infty$，则
$$
E[Y]=E[g(x)]=\sum_{k=1}^{\infty} g(x_k)p_k
$$
若 $X$ 为连续型随机变量，密度为 $p(x)$，且期望存在，则
$$
E[Y]=E[g(X)]=\int_{-\infty}^{+\infty}g(x)p(x)dx
$$

> 证明：这里仅讨论离散情形的证明。设 $Y=g(X)$ 的取值为 $y_1,y_2,\cdots$，$Y$ 显然也是离散型随机变量。记 $A_n=\{X的取值\}\cap \{x|g(x)=y_n\}=\{x_n^1,x_n^2,\cdots\}$，则 $A$ 构成了全空间 $\Omega$ 的一个划分。
> $$
> \begin{align}
> E[Y]&=\sum_{n=1}^\infty y_nP(X\in A_n)=\sum_{n=1}^\infty y_n\sum_{m=1}^\infty P(X=x_n^m)\\\\
> &=\sum_{n=1}^\infty \sum_{m=1}^\infty y_nP(X=x_n^m)=\sum_{n=1}^\infty\sum_{m=1}^\infty g(x_n^m)P(X=x_n^m)\\\\
> &=\sum_{n=1}^\infty \sum_{x\in A_n}g(x)P(X=x)\\\\
> &=\sum_{k=1}^\infty g(x_k)P(X=x_k)\qquad \blacksquare
> \end{align}
> $$

对于二维随机变量函数我们也有类似的结论：

$\fbox{Theorem 13.4}$ 设 $(X,Y)$ 为二维随机变量，$Z=g(X,Y)$。

(1) 离散情形：$(X,Y)$ 有分布律 $P(X=x_i,Y=y_j)=p_{i,j}$。若期望存在，那么
$$
E[Z]=E[g(X,Y)]=\sum_{i=1}^\infty\sum_{j=1}^\infty g(x_i,y_j)p_{i,j}
$$
(2) 连续情形：$(X,Y)$ 的密度函数为 $p(x,y)$，若期望存在，则
$$
E[Z]=E[g(X,Y)]=\int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty}g(x,y)p(x,y)dxdy
$$
【例题】 (习题 4.5) 商品出口需求 $X\sim U[2000,4000]$，售出 1 单位商品可以获得 3 万元收益，不能售出要倒贴  1 万元。问应出口多少吨才能得到最大收益。

> 解：设出口量为 $y$，显然有 $2000\leq y\leq 4000$。令收益为 $Z$：
> $$
> Z=g(X)=\begin{cases}3y&,X>y\\\\3X-(y-X)&,X\leq y\end{cases}
> $$
> 考虑 $Z$ 的期望：
> $$
> \begin{align}
> E[Z]&=E[g(X)]=\int_{-\infty}^{+\infty}g(x)p(x)dx=\int_{2000}^{4000}g(x)\frac{1}{2000}dx\\\\
> &=\frac{1}{2000}\left(\int_{2000}^y(4x-y)dx+\int_y^{4000}3ydx\right)\\\\
> &=\frac{1}{1000}(-y^2+7000y-4\cdot 10^6)
> \end{align}
> $$
> 当 $y=3500$ 时期望收益最大。

## Properties of Expectation

* 若随机变量 $X\equiv a$，则 $E\[x\]=a$。

* 对于任意常数 $a,b$，有 $E\[aX+bY\]=aE\[X\]+bE\[Y\]$。

    > 证明：这里的证明考虑 $X,Y$ 是离散型随机变量的情况：
    >
    > 设 $X,Y$ 的分布律为 $P(X=x_i,Y=y_j)=p_{ij}$，那么
    > $$
    > \begin{align}
    > E[aX+bY]&=\sum_{i=1}^\infty\sum_{j=1}^\infty(ax_i+by_j)p_{ij}\\\\
    > &=a\sum_{i=1}^\infty x_i\sum_{j=1}^\infty p_{ij}+b\sum_{j=1}^\infty y_j\sum_{i=1}^\infty p_{ij}\\\\
    > &=a\sum_{i=1}^\infty x_ip_{i\cdot}+b\sum_{j=1}^\infty y_jp_{\cdot j}\\\\
    > &=aE[X]+bE[Y]
    > \end{align}
    > $$

* 若 $X,Y$ 独立，则 $E\[XY\]=E\[X\]E\[Y\]$。

    > 证明：这里的证明考虑 $X,Y$ 是离散型随机变量的情况：
    >
    > 设 $X,Y$ 的分布律为 $P(X=x_i,Y=y_j)=p_{ij}$，因为 $X,Y$ 相互独立，所以 $p_{ij}=p_{i\cdot}\cdot p_{\cdot j}$。那么
    > $$
    > \begin{align}
    > E(XY)&=\sum_{i=1}^\infty\sum_{j=1}^\infty x_iy_jp_{ij}\\\\
    > &=\sum_{i=1}^\infty\sum_{j=1}^\infty x_iy_jp_{i\cdot }p_{\cdot j}\\\\
    > &=\left(\sum_{i=1}^\infty x_ip_{i\cdot }\right)\left(\sum_{j=1}^\infty y_jp_{\cdot j}\right)\\\\
    > &=E[X]E[Y]
    > \end{align}
    > $$

利用期望的线性性，我们可以很方便地解决一些问题，比如考虑 $X\sim B(n,p)$ 的期望，令 $x_i$ 表示第 $i$ 次实验是否成功的示性随机变量 i.e. $x_i=\begin{cases}1&,第i次实验成功\\\\0&,otherwise\end{cases}$，容易发现 $E(x_i)=P(x_i=1)=p$。那么
$$
E[X]=E\left[\sum_{i=1}^nx_i\right]=\sum_{i=1}^nE[x_i]=\sum_{i=1}^nP(x_i=1)=np.
$$
再举一例，设 $X$ 服从超几何分布，考虑 $X$ 的期望。类似地，我们令示性随机变量 $x_i$ 刻画第 $i$ 个次品被抽中的情况，显然有 $P(x_i=1)=\frac{\binom{N-1}{n-1}}{\binom{N}{n}}=\frac{n}{N}$ (分子的意义是：保证该次品被抽中，剩下的 $n-1$ 个物品随便抽)，那么
$$
E[x]=E\left[\sum_{i=1}^Mx_i\right]=\sum_{i=1}^ME[x_i]=\sum_{i=1}^M\frac{n}{N}=\frac{nM}{N}.
$$
【例题】 (习题 4.8) $n$ 个人无放回拿 $n$ 个帽子，求拿到自己的帽子的人数的期望。

> 解：令 $x_i$ 是刻画第 $i$ 个人是否拿到帽子的示性随机变量，那么
> $$
> E[X]=E\left[\sum_{i=1}^nx_i\right]=\sum_{i=1}^nE[x_i]=\sum_{i=1}^n\frac{(n-1)!}{n!}=1.
> $$