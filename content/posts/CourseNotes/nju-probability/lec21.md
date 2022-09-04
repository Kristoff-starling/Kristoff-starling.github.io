---
title: "Lecture 21: Interval Estimation"
linktitle: Lecture 21
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
        weight: 21

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

$\fbox{Definition 21.1}$ 设 $\theta$ 是总体 $X$ 的未知参数，$X_1,\cdots, X_n$ 是来自 $X$ 的样本，若对于给定的 $0<\alpha<1$，存在两个统计量 $\hat\theta_1(X_1,\cdots, X_n)$ 和 $\hat\theta_2(X_1,\cdots, X_n)$，使得
$$
P(\hat\theta_1<\theta<\hat\theta_2)=1-\alpha
$$
则称 $(\hat\theta_1,\hat\theta_2)$ 是 $\theta$ 的置信度为 $1-\alpha$ 的置信区间，$\theta_1,\theta_2$ 称为置信下限和置信上限，$1-\alpha$ 称为置信度或置信系数。

注：(1) 上面数学表达式的含义为：在进行了 $m$ 轮 (每轮 $n$ 次) 采样后，我们会得到 $m$ 个区间 $(\hat\theta_{1,1},\hat\theta_{2,1})$ …… $(\hat\theta_{1,m},\hat\theta_{2,m})$。这其中大约有 $(1-\alpha)m$ 个区间包含了 $\theta$。对于一组具体的估计值 $\hat\theta_1(x_1,\cdots,x_n)$ 和 $\hat\theta_2(x_1,\cdots, x_n)$，我们不能说 $\theta$ 有 $1-\alpha$ 的概率落在区间中，因为这时区间的上下界已经是确定的数了，不再具有随机性。

(2) $\hat\theta_2-\hat\theta_1$ 越大，置信度越高，但精度也会越差。

有时我们也会只关心未知参数的上限或下限 (如保质期)，因此我们类似地有单侧置信区间和单侧置信上限/下限的概念。

## Pivot Method

枢轴变量法的主要思想如下：

* 构造一个样本函数 $U(X_1,\cdots, X_n;\theta)$，我们要求 $U$ 是有且仅有未知参数 $\theta$ 的函数，且 $U$ 的分布已知。我们称 $U$ 为枢轴变量。
* 因为 $U$ 的分布已知，所以对于给定的置信区间 $1-\alpha$，我们可以确定区间 $[a,b]$，使得 $P(a\leq U\leq b)=1-\alpha$ (如果要求单边置信区间，则只需要一个方向的约束)。
* 根据 $a\leq U\leq b$ 反推出 $\theta$ 的范围 (范围表达式中仅包含 $X_1,\cdots, X_n$)，这个范围就是我们要求的 $\hat\theta_1$ 和 $\hat\theta_2$。

## Confidence Interval of $\mu$ in Normal Population

【例】给定置信度 $1-\alpha$，样本 $X_1,\cdots, X_n$ 来自总体 $X\sim N(\mu,\sigma^2)$。考虑用枢轴变量法求均值 $\mu$ 的置信区间：因为 $X\sim N(\mu,\sigma^2)$，所以 $\overline{X}\sim N(\mu,\frac{\sigma^2}{n})$ 我们构造 $U=\frac{\overline{X}-\mu}{\sigma/\sqrt n}$，则 $U\sim N(0,1)$。

记 $u_{\alpha}$ 是 $N(0,1)$ 的上 $\alpha$ 分为数 (即 $P(u>u_\alpha)=\alpha$)，那么
$$
1-\alpha=P(u_{1-\alpha/2}<U<u_{\alpha/2})=P(-u_{\alpha/2}<U<u_{\alpha/2})
$$
因此
$$
-u_{\frac{\alpha}{2}}<\frac{\overline{X}-\mu}{\sigma/\sqrt n}<u_{\frac{\alpha}{2}}\quad \Rightarrow\quad \overline{X}-\frac{\sigma u_{\frac{\alpha}{2}}}{\sqrt n}<\mu<\overline{X}+\frac{\sigma \mu_{\frac{\alpha}{2}}}{\sqrt n}
$$
因此置信区间为 $\left(\overline{X}-\frac{\sigma u_{\frac{\alpha}{2}}}{\sqrt n},\overline{X}+\frac{\sigma \mu_{\frac{\alpha}{2}}}{\sqrt n}\right)$。

如果我们不知道 $\sigma^2$，我们也可以给出一个置信区间。根据正态总体章节的知识，我们有
$$
T=\frac{\sqrt n(\overline{X}-\mu)}{S}\sim t(n-1)
$$
因此记 $t_{\alpha}(n-1)$ 为 $T$ 的上 $\alpha$ 分为数，有
$$
1-\alpha=P(t_{1-\alpha/2}(n-1)<T<t_{\alpha/2}(n-1))=P(-t_{\alpha/2}(n-1)<T<t_{\alpha/2}(n-1))
$$
因此
$$
-t_{\frac{\alpha}{2}}(n-1)<\frac{\sqrt n(\overline{X}-\mu)}{S}<t_{\frac{\alpha}{2}}(n-1)\quad \Rightarrow \quad 置信区间\left(\overline{X}-\frac{St_{\frac{\alpha}{2}}(n-1)}{\sqrt n},\overline{X}+\frac{St_{\frac{\alpha}{2}}(n-1)}{\sqrt n}\right)
$$
$T$ 分布的尾部比正态分布要重 (正态分布是指数衰减的，T 分布是多项式衰减的)，因此上述置信区间会比知道 $\sigma^2$ 时的置信区间要宽一些。考虑到我们是在知道更少的信息的情况下求出的该区间，这一点是容易理解的。

## Confidence Interval of $\sigma^2$ in Normal Population

【例】给定置信度 $1-\alpha$，样本 $X_1,\cdots, X_n$ 来自总体 $X\sim N(\mu,\sigma^2)$。考虑用枢轴变量法求方差 $\sigma^2$ 的置信区间：在 $\mu$ 未知的情况下，我们可以这样给出枢轴变量：$\frac{(n-1)S^2}{\sigma^2}\sim \chi^2(n-1)$。从而
$$
1-\alpha=P\left(\chi^2_{1-\frac{\alpha}{2}}(n-1)<\frac{(n-1)S^2}{\sigma^2}<\chi^2_{\frac{\alpha}{2}}(n-1)\right)
$$
解得置信区间为
$$
\left(\frac{(n-1)S^2}{\chi^2_{\frac{\alpha}{2}}(n-1)},\frac{(n-1)S^2}{\chi^2_{1-\frac{\alpha}{2}}(n-1)}\right)
$$
注：(1) 尽管 $\chi^2$ 分布和 $F$ 分布非对称，但通常仍然沿用 $\alpha/2$ 和 $1-\alpha/2$ 这两个分位点。这样求得的置信区间可能并非最短。

(2) 若 $\mu$ 已知，我们可以将其运用起来精确化我们的置信区间：取 $U=\left(\frac{\overline{X}-\mu}{\sigma/\sqrt n}\right)^2$，则 $U\sim \chi^2(1)$。

## Confidence Interval of $\mu_1-\mu_2$ in Normal Population

【例】给定置信度 $1-\alpha$，样本 $X_1,\cdots, X_{n_1}$ 来自总体 $X\sim N(\mu_1,\sigma_1^2)$，$Y_1,\cdots, Y_{n_2}$ 来自总体 $Y\sim N(\mu_2,\sigma_2^2)$，考虑它们的均值差 $\mu_1-\mu_2$ 的置信区间。如果 $\sigma_1^2$ 和 $\sigma_2^2$ 均已知，那么 $\overline{X}\sim N(\mu_1,\frac{\sigma_1^2}{n_1})$，$\overline{Y}\sim N(\mu_2,\frac{\sigma_2^2}{n_2})$，$\overline{X}-\overline{Y}\sim N(\mu_1-\mu_2,\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2})$。因此取
$$
U=\frac{\overline{X}-\overline{Y}-(\mu_1-\mu_2)}{\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}}\sim N(0,1)
$$
从而
$$
-u_{\frac{\alpha}{2}}<\frac{\overline{X}-\overline{Y}-(\mu_1-\mu_2)}{\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}}<u_{\frac{\alpha}{2}}\Rightarrow置信区间\left(\overline{X}-\overline{Y}-\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}u_{\frac{\alpha}{2}},\overline{X}-\overline{Y}+\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}u_{\frac{\alpha}{2}}\right)
$$
若不知道 $\sigma_1^2$ 和 $\sigma_2^2$，但知道 $\sigma_1^2=\sigma_2^2=\sigma^2$，那么考虑如下公式：
$$
T=\sqrt{\frac{n_1n_2(n_1+n_2-2)}{n_1+n_2}}\frac{(\overline X-\overline Y)-(\mu_1-\mu_2)}{\sqrt{(n_1-1)S_X^2+(n_2-1)S_Y^2}}\sim t(n_1+n_2-2)
$$
于是有
$$
1-\alpha=P(-t_{\alpha/2}(n_1+n_2-2)<T<t_{\alpha/2}(n_1+n_2-2))
$$
令 $C_{n_1,n_2}=\sqrt{\frac{n_1n_2(n_1+n_2-2)}{n_1+n_2}}\frac{1}{\sqrt{(n_1-1)S_X^2+(n_2-1)S_Y^2}}$，则
$$
-t_{\frac{\alpha}{2}}(n_1+n_2-2)<C_{n_1,n_2}[(\overline X-\overline Y)-(\mu_1-\mu_2)]<t_{\frac{\alpha}{2}}(n_1+n_2-2)
$$
解得置信区间为
$$
\left(\overline{X}-\overline{Y}-\frac{t_{\frac{\alpha}{2}}(n_1+n_2-2)}{C_{n_1,n_2}},\overline{X}-\overline{Y}+\frac{t_{\frac{\alpha}{2}}(n_1+n_2-2)}{C_{n_1,n_2}}\right)
$$

## Confidence Interval of $\frac{\sigma_1^2}{\sigma_2^2}$ in Normal Population

【例】给定置信度 $1-\alpha$，样本 $X_1,\cdots, X_{n_1}$ 来自总体 $X\sim N(\mu_1,\sigma_1^2)$，$Y_1,\cdots, Y_{n_2}$ 来自总体 $Y\sim N(\mu_2,\sigma_2^2)$，考虑它们的方差比 $\frac{\sigma_1^2}{\sigma_2^2}$ 的置信区间。若 $\mu_1,\mu_2$ 未知，我们考虑枢轴变量
$$
F=\frac{S_X^2\sigma_2^2}{S_Y^2\sigma_1^2}\sim F(n_1-1,n_2-2)
$$
从而
$$
1-\alpha=P(F_{1-\frac{\alpha}{2}}(n_1-1,n_2-1)<F<F_{\frac{\alpha}{2}}(n_1-1,n_2-1))
$$
解得置信区间为
$$
\left(\frac{S_X^2}{S_Y^2}\frac{1}{F_\frac{\alpha}{2}(n_1-1,n_2-1)},\frac{S_X^2}{S_Y^2}\frac{1}{F_{1-\frac{\alpha}{2}}(n_1-1,n_2-1)}\right)
$$
如果我们知道更多信息，则可以给出更好的估计，比如：

* 若 $\mu_1$ 已知，$\mu_2$ 未知，那么 $\frac{\overline X-\mu_1}{\sigma_1/\sqrt{n_1}}\sim N(0,1)$，$\frac{(n_2-1)S_Y^2}{\sigma_2^2}\sim \chi^2(n_2-1)$，则
    $$
    U=\frac{\left(\frac{\overline X-\mu_1}{\sigma_1/\sqrt{n_1}}\right)^2}{S_Y^2/\sigma_2^2}\sim F(1,n_2-1)
    $$
    (注：分子正态分布的平方，即一个自由度的 $\chi^2$ 分布。)

* 若 $\mu_1,\mu_2$ 均已知，则
    $$
    U=\frac{\left(\frac{\overline X-\mu_1}{\sigma_1/\sqrt{n_1}}\right)^2}{\left(\frac{\overline Y-\mu_2}{\sigma_2/\sqrt{n_2}}\right)^2}\sim F(1,1)
    $$

## Interval Estimation of Unknown Population

非正态总体均值的区间估计通常采用所为的大样本法：

设 $X_1,\cdots, X_n$ 来自总体 $X$，且 $E[X]=\mu,D(X)=\sigma$，考虑计算 $\mu$ 的置信度为 $1-\alpha$ 的置信区间。由中心极限定理，有
$$
\frac{\sum_{k=1}^nX_k-n\mu}{\sqrt n\sigma}\to N(0,1)
$$
从而考虑 $U=\frac{\overline{X}-\mu}{\sigma/\sqrt n}\approx N(0,1)$，于是 $1-\alpha\approx P(-u_{\frac{\alpha}{2}}<U<u_{\frac{\alpha}{2}})$。

* 当 $\sigma^2$ 已知时，置信区间为 $(\overline X-\frac{u_{\frac{\alpha}{2}}\sigma}{\sqrt n},\overline X+\frac{u_{\frac{\alpha}{2}}\sigma}{\sqrt n})$。

* 当 $\sigma^2$ 未知时，用 $S$ 来代替 $\sigma$。

【例题】设总体服从 0-1 分布，$X_1,\cdots, X_n$ 为样本，求总体期望 $p$ 的置信度为 $1-\alpha$ 的置信区间。

> 解法一：该题设符合方差未知的情形，所以使用第二个结论，所求区间为
> $$
> \left(\overline X-\frac{u_{\frac{\alpha}{2}}S}{\sqrt n},\overline X+\frac{u_{\frac{\alpha}{2}}S}{\sqrt n}\right)
> $$
> 这里的 $S$ 是一个二阶的统计量，利用 0-1 分布的性质，我们可以进一步化简它：
> $$
> \begin{align}
> S^2&=\frac{1}{n-1}\sum_{k=1}^n(X_k-\overline{X})^2=\frac{1}{n-1}\left(\sum_{k=1}^nX_k^2-n\overline{X}^2\right)\\\\
> &\overset{\text{0-1分布}}{=}\frac{1}{n-1}\left(\sum_{k=1}^nX_k-n\overline{X}^2\right)\\\\
> &=\frac{n}{n-1}\overline{X}(1-\overline{X})
> \end{align}
> $$
> 代入原式得置信区间：
> $$
> \left(\overline{X}-u_{\frac{\alpha}{2}}\sqrt{\frac{\overline{X}(1-\overline X)}{n-1}},\overline{X}+u_{\frac{\alpha}{2}}\sqrt{\frac{\overline{X}(1-\overline X)}{n-1}}\right)
> $$
> 这样置信区间中只有一阶的统计量，更加准确。

> 解法二：因为 $\sum_{k=1}^nX_k\sim B(n,p)$，伯努利试验中方差 $\sigma=\sqrt{p(1-p)}$。根据中心极限定理，
> $$
> \frac{\sum_{k=1}^nX_k-np}{\sqrt{n}\sqrt{p(1-p)}}\to N(0,1)
> $$
> 因此可以得到近似的置信区间
> $$
> \left(\overline{X}-u_{\frac{\alpha}{2}}\sqrt{\frac{p(1-p)}{n}},\overline{X}+u_{\frac{\alpha}{2}}\sqrt{\frac{p(1-p)}{n}}\right)
> $$
> $\overline{X}$ 是对 $p$ 的一个较好的估计，因此用 $\overline{X}$ 代替 $p$，即得到
> $$
> \left(\overline{X}-u_{\frac{\alpha}{2}}\sqrt{\frac{\overline{X}(1-\overline{X})}{n}},\overline{X}+u_{\frac{\alpha}{2}}\sqrt{\frac{\overline{X}(1-\overline{X})}{n}}\right)
> $$
> 该做法更好地利用了 0-1 分布 (伯努利试验)，因此给出的区间更加精确一些。