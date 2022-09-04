---
title: "Lecture 22: Hypothesis Test"
linktitle: Lecture 22
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
        weight: 22

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

设总体 $X\sim N(\mu,\sigma^2)$，欲判断 $\mu$ 是否为某个给定的常数 $\mu_0$，我们将 $\mu=\mu_0$ ，记为 $H_0:\mu=\mu_0$，称为原假设或零假设。对应的，$H_1:\mu\neq \mu_0$ 称为备择假设或对立假设。
$$
H_0:\mu=\mu_0\qquad H_1:\mu\neq \mu_0
$$
我们也可以考虑判断 $\mu\leq \mu_0$ 是否成立，这时的原假设和对立假设为
$$
H_0:\mu\leq \mu_0\qquad H_1:\mu>\mu_0
$$
第一种假设称为双边检验 (对立假设居于原假设两边)，第二种则是单边检验。上述两种假设只涉及已知总体的未知参数，称为参数假设检验 (像 $H_0:X服从正态分布$ 就是非参数假设检验)。

以第一种假设为例介绍假设检验的一般方法：$\overline{X}$ 是 $\mu$ 的一个比较好的估计，因此 $|\overline{X}-\mu_0|$ 应该不太大。我们希望找到一个阈值 $k$，当 $|\overline{X}-\mu_0|< k$ 时，我们认为假设成立。问题的关键在于如何选取 $k$。在 $H_0$ 成立的情况下，$\overline{X}\sim N(\mu_0,\frac{\sigma^2}{n})$。考虑 $\sigma^2$ 已知的情形，则 $U=\frac{\overline{X}-\mu_0}{\sigma/\sqrt n}\sim N(0,1)$。$|\overline{X}-\mu_0|\geq k$ 意味着 $|U|\geq \frac{k}{\sigma/\sqrt n}$，故 $k$ 的选取应当使得 $P\left(|U|\geq \frac{k}{\sigma/\sqrt n}\right)$ 足够小。给定一个小概率 $\alpha$ (常用值：0.1, 0.05, 0.01)，令 $P(|U|\geq \frac{k}{\sigma/\sqrt n})=\alpha$，即可确定 $\frac{k}{\sigma/\sqrt n}=u_{\frac{\alpha}{2}}$。

我们把 $U$ 称为检验统计量，$\alpha$ 称为显著性水平，$u_{\frac{\alpha}{2}}=\frac{k}{\sigma/\sqrt n}$ 称为临界值。$W=\{|U|\geq u_{\frac{\alpha}{2}}\}$ 称为拒绝域，$\overline{W}$ 称为接受域。

基本步骤：

* 根据问题提出 $H_0$ 和 $H_1$。
* 构造一个合适的统计量，在 $H_0$ 成立的条件下求该统计量的分布。
* 给出小概率 $\alpha$，确定临界值和拒绝域 $W$。
* 由样本算出观察值，若其落入 $W$，则拒绝 $H_0$，否则接受 $H_0$。

假设检验的两类错误：

* 弃真错误：$H_0$ 正确却拒绝了 $H_0$。$P(拒绝H_0|H_0为真)=P(U\in W|H_0为真)=\alpha$。
* 存伪错误：$H_0$ 错误却接受了 $H_0$。$P(接受H_0|H_1为真)=P(U\notin W|H_1为真)=\beta$。

通常来说 $\beta$ 不容易求得。$\alpha$ 和 $\beta$ 不能同时减小，我们通常在控制 $\alpha$ 足够小的情况下尽可能减小 $\beta$ (Neyman-Pearson 原则)，即拒绝 $H_0$ 需要更充分的理由，$H_0$ 地位高于 $H_1$。

【例题】设 $X\sim N(\mu,1)$，$X_1,\cdots, X_n$ 为样本。在给定显著性水平 $\alpha$ 的情况下求 $H_0:\mu=\mu_0$，$H_1:\mu=\mu_1$ 的第二类错误概率 $\beta$。

> 解：在 $H_0$ 下，我们会构造 $U=\frac{\overline X-\mu_0}{1/\sqrt n}=\sqrt n(\overline X-\mu_0)\sim N(0,1)$。但现在 $H_1$ 成立，因此 $U$ 实际服从的分布为
> $$
> U=\sqrt n(\overline X-\mu_0)=\sqrt n(\overline X-\mu_1)+\sqrt n(\mu_1-\mu_0)\sim N(\sqrt n(\mu_1-\mu_0),1)
> $$
> 根据 $\beta$ 的定义，
> $$
> \begin{align}
> \beta&=P(U\notin W|H_1为真)=P(|U|<u_{\frac{\alpha}{2}}|\mu=\mu_1)\\\\
> &=P(-u_{\frac{\alpha}{2}}-\sqrt n(\mu_1-\mu_0)<\sqrt n(\overline{X}-\mu_1)<u_{\frac{\alpha}{2}}-\sqrt n(\mu_1-\mu_0)|\mu=\mu_1)\\\\
> &=\Phi(u_{\frac{\alpha}{2}}-\sqrt n(\mu_1-\mu_0))-\Phi(-u_{\frac{\alpha}{2}}-\sqrt n(\mu_1-\mu_0))
> \end{align}
> $$

## Mean Value Test of Normal Population

总体 $X\sim N(\mu,\sigma^2)$，$X_1,\cdots,X_n$ 为样本，给定 $\alpha$。

### $u$ Test

在 $\sigma^2$ 已知的情况下，$H_0:\mu=\mu_0,H_1:\mu\neq \mu_0$。考虑 $U=\frac{\overline{X}-\mu}{\sigma/\sqrt n}\sim N(0,1)$。容易得出临界值 $P(|U|>u_{\frac{\alpha}{2}})=\alpha$，拒绝域 $W=\left\\{\left|\frac{\overline{X}-\mu_0}{\sigma/\sqrt n}\right|>u_{\frac{\alpha}{2}}\right\\}$。

如果做单边检验：$H_0:\mu=\mu_0,H_1:\mu>\mu_0$，则临界值为 $u_\alpha$，拒绝域：$W=\left\\{\frac{\overline{X}-\mu_0}{\sigma/\sqrt n}>u_\alpha\right\\}$。

若假设为 $H_0:\mu\leq \mu_0,H_1:\mu>\mu_0$。我们的拒绝域 $W$ 应该满足 $P(U\in W|H_0)\leq \alpha$。我们发现 $\mu_0$ 是 $\mu$ 的上界，因此
$$
P\left(\left.\frac{\overline{X}-\mu_0}{\sigma/\sqrt n}>\lambda_\alpha\right|\mu\leq \mu_0\right)\leq P\left(\left.\frac{\overline{X}-\mu}{\sigma/\sqrt n}>\lambda_\alpha\right|\mu\leq \mu_0\right)
$$
令右式等于 $\alpha$，因为 $\frac{\overline{X}-\mu}{\sigma/\sqrt n}\sim N(0,1)$，所以解得 $\lambda_\alpha=u_\alpha$。拒绝域：$W=\left\\{\frac{\overline{X}-\mu_0}{\sigma/\sqrt n}>u_\alpha\right\\}$。

### $t$ Test

在 $\sigma^2$ 未知的情况下，$H_0:\mu=\mu_0,H_1:\mu\neq \mu_0$，取
$$
T=\frac{\sqrt n(\overline{X}-\mu_0)}{S}\sim t(n-1)
$$
可解出拒绝域 $W=\left\\{|T|>t_{\frac{\alpha}{2}}(n-1)\right\\}$。

## Test of the Difference of Mean Values of Two Normal Populations

总体 $X\sim N(\mu_1,\sigma_1^2),Y\sim N(\mu_2,\sigma_2^2)$。假设 $H_0:\mu_1=\mu_2,H_1:\mu_1\neq \mu_2$。考虑以下几种情况：

(1) $\sigma_1^2,\sigma_2^2$ 已知，取
$$
U=\frac{\overline X-\overline Y-(\mu_1-\mu_2)}{\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}}\overset{在H_0下}{=}\frac{\overline X-\overline Y}{\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}}\sim N(0,1)
$$
因此拒绝域
$$
W=\left\\{\left|\frac{\overline X-\overline Y}{\sqrt{\frac{\sigma_1^2}{n_1}+\frac{\sigma_2^2}{n_2}}}\right|>u_{\frac{\alpha}{2}}\right\\}
$$
(2) $\sigma_1^2=\sigma_2^2=\sigma^2$ 但未知，取
$$
T=\sqrt{\frac{n_1n_2(n_1+n_2-2)}{n_1+n_2}}\frac{(\overline X-\overline Y)-(\mu_1-\mu_2)}{\sqrt{(n_1-1)S_X^2+(n_2-1)S_Y^2}}\sim t(n_1+n_2-2)
$$
(类似地，在 $H_0$ 下，$\mu_1-\mu_2$ 可以消去)

因此拒绝域
$$
W=\left\\{|T|>t_{\frac{\alpha}{2}}(n_1+n_2-2)\right\\}
$$

### Pairwise $t$ Test

另外一种情况是，$X,Y$ 分布未知且相关，但 $Z=X-Y\sim N(\mu,\sigma^2)$。我们想要检验这两个分布是否接近。假设 $H_0:\mu=0,H_1:\mu\neq 0$。那么取
$$
T=\frac{\sqrt n(\overline{Z}-\mu)}{S_Z}\overset{在H_0下}{=}\frac{\sqrt n\overline{Z}}{S_Z}\sim t(n-1)
$$
拒绝域
$$
W=\left\\{|T|>t_{\frac{\alpha}{2}}(n-1)\right\\}
$$

## Variance Test of Normal Population

设 $X\sim N(\mu,\sigma^2)$，$X_1,\cdots, X_n$ 来自总体 $X$，$\alpha$ 为显著水平。若 $\mu$ 未知，假设 $H_0:\sigma^2=\sigma_0^2,H_1:\sigma^2\neq \sigma_0^2$，考虑变量
$$
\chi^2=\frac{(n-1)S^2}{\sigma_0^2}\sim \chi^2(n-1)
$$
因此拒绝域
$$
W=\{\chi^2<\chi_{1-\frac{\alpha}{2}}^2(n-1)\}\cup\{\chi^2>\chi^2_{\frac{\alpha}{2}}(n-1)\}
$$
两正态总体方差比值的检验：设 $X\sim N(\mu,\sigma^2)$，$X_1,\cdots, X_n$ 来自总体 $X$，$\alpha$ 为显著水平。这次利用 F 分布
$$
F=\frac{S_1^2\sigma_2^2}{S_2^2\sigma_1^2}\sim F(n_1-1,n_2-2)
$$
因此拒绝域
$$
W=\{F<F_{1-\frac{\alpha}{2}}(n_1-1,n_2-1)\}\cup \{F>F_{\frac{\alpha}{2}}(n_1-1,n_2-1)\}
$$