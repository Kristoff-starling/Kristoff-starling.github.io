---
title: "Moment Generating Function"
linktitle: MGF
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
        parent: Materials
        weight: 1

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

矩母函数是快速求解高阶矩的利器。

## Definitions and Properties

$\fbox{Definition 1}$ ($k$ 阶矩) 设 $X$ 是一个概率密度为 $f$ 的随机变量。若 $X$ 是离散的，则它的 **$k$ 阶矩** (记作 $\mu_k$) 定义为
$$
\mu_k\triangleq \sum_{m=0}^{\infty}x_m^kf(x_m)
$$
若 $X$ 是连续的，则它的 $k$ 阶矩定义为
$$
\mu_k\triangleq \int_{-\infty}^{+\infty}x^kf(x)\text{d}x
$$
$\fbox{Definition 2}$ (矩母函数) 设 $X$ 是一个概率密度为 $f$ 的随机变量，则其**矩母函数** $M_X(t)$ 定义为 $\mathbb E[e^{tX}]$。具体地说，若 $X$ 是离散的，则
$$
M_X(t)\triangleq \sum_{m=0}^\infty e^{tx_m}f(x_m)
$$
若 $X$ 是连续的，则
$$
M_X(t)\triangleq \int_{-\infty}^{+\infty}e^{tx}f(x)\text{d}x
$$
矩母函数有如下重要性质：

* $$
    M_X(t)=\sum_{k=0}^\infty\mu_k\frac{t^k}{k!}
    $$

    > 证明：此处仅证明连续情形。考虑将矩母函数中的 $e^{tx}$ 泰勒展开，则有
    > $$
    > \begin{align}
    > M_X(t)&=\int_{-\infty}^{+\infty}e^{tx}f(x)\text{d}x\\\\
    > &=\int_{-\infty}^{+\infty}\left(\sum_{k=0}^\infty\frac{(tx)^k}{k!}\right)f(x)\text{d}x\\\\
    > &=\sum_{k=0}^\infty\frac{t^k}{k!}\int_{-\infty}^{+\infty}x^kf(x)\text{d}x\\\\
    > &=\sum_{k=0}^\infty\frac{t^k}{k!}\mu_k\qquad \blacksquare
    > \end{align}
    > $$

    因此我们可以通过对矩母函数求导的方式来快速获得一个随机变量的各个 $k$ 阶矩：
    $$
    \mu_k=\left.\frac{\text{d}^kM_X(t)}{\text{d}x^k}\right|_{t=0}
    $$

* 设 $\alpha$ 和 $\beta$ 为常数，那么
    $$
    M_{\alpha X+\beta}(t)=e^{\beta t}M_X(\alpha t)
    $$

    > 证明：
    > $$
    > \begin{align}
    > M_{\alpha X+\beta}(t)=\mathbb E[e^{t(\alpha X+\beta)}]=e^{\beta t}\mathbb E[e^{\alpha t\cdot X}]=e^{\beta t}M_X(\alpha t)\qquad \blacksquare
    > \end{align}
    > $$

* 设 $X_1$ 和 $X_2$ 为互相独立的两个随机变量，且 $|t|<\delta$ 时 $M_{X_1}(t)$ 和 $M_{X_2}(t)$ 均收敛，那么
    $$
    M_{X_1+X_2}(t)=M_{X_1}(t)\cdot M_{X_2}(t)
    $$

    > 证明：
    > $$
    > \begin{align}
    > M_{X_1+X_2}(t)=\mathbb E[e^{t(X_1+X_2)}]=\mathbb E[e^{tX_1}\cdot e^{tX_2}]=\mathbb E[e^{tX_1}]\cdot \mathbb E[e^{tX_2}]=M_{X_1}(t)\cdot M_{X_2}(t)\qquad \blacksquare
    > \end{align}
    > $$

## MGF of Common Distribution

### Bernoulli Distribution

设 $X\sim B(n,p)$，即 $P(X=k)=f(k)=\binom{n}{k}p^k(1-p)^{n-k}$，那么
$$
\begin{align}
M_X(t)&=\sum_{k=0}^\infty e^{tk}f(k)\\\\
&=\sum_{k=0}^n e^{tk}\binom{n}{k}p^k(1-p)^{n-k}\\\\
&=\sum_{k=0}^n\binom{n}{k}(e^tp)^k(1-p)^{n-k}\\\\
&=(e^tp+1-p)^n
\end{align}
$$

### Poisson Distribution

设 $X\sim p(\lambda)$，即 $P(X=k)=f(k)=\frac{\lambda^ke^{-\lambda}}{k!}$，那么
$$
\begin{align}
M_X(t)&=\sum_{k=0}^\infty e^{tk}f(k)\\\\
&=\sum_{k=0}^\infty e^{tk}\cdot \frac{\lambda^ke^{-\lambda}}{k!}\\\\
&=e^{-\lambda}\sum_{k=0}^\infty\frac{(\lambda e^t)^k}{k!}\\\\
&=e^{-\lambda}\cdot e^{\lambda e^t}=e^{\lambda(e^t-1)}
\end{align}
$$

### Normal Distribution

我们首先考虑标准正态分布：设 $Y\sim N(0,1)$，那么 $Y$ 的密度函数 $f(y)=\frac{1}{\sqrt{2\pi}}e^{-\frac{y^2}{2}}$，从而
$$
\begin{align}
M_Y(t)&=\mathbb E[e^{tY}]=\int_{-\infty}^{+\infty}e^{ty}f(y)\text{d}y\\\\
&=\frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}e^{-\frac{y^2}{2}+ty}\text{d}y\\\\
&=e^{\frac{1}{2}t^2}\cdot \frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}e^{-\frac{1}{2}(y-t)^2}\text{d}y
\end{align}
$$
注意到后半部分恰好是 $N(t,1)$ 的概率密度函数在 $\mathbb R$ 上的积分，因此后半部分为 1，从而 $M_Y(t)=e^{\frac{1}{2}t^2}$。

对于一般的正态分布 $X\sim N(\mu,\sigma^2)$，注意到 $\frac{X-\mu}{\sigma}=Y$，即 $X=\sigma Y+\mu$，因此根据矩母函数的性质，
$$
M_X(t)=M_{\sigma Y+\mu}(t)=e^{\mu t}M_Y(\sigma t)=e^{\mu t+\frac{1}{2}\sigma^2t^2}
$$
正态分布的矩母函数恰好就是对数正态分布的 $k$ 阶矩，推导过程如下：假设 $X$ 服从对数正态分布，那么 $\ln X\sim N(\mu,\sigma)$，从而
$$
\mathbb E[X^k]=\mathbb E[(e^{\ln X})^k]=\mathbb E[e^{k\ln X}]=M_{\ln X}(k)=e^{\mu k+\frac{1}{2}\sigma^2k^2}
$$
注：对数正态分布没有矩母函数。

