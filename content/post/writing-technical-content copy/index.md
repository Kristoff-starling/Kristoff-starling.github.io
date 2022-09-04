---
title: "Lecture : Criterion of Estimator"
date: 2019-07-12
math: true
---

## Unbiasedness

$\fbox{Definition 20.1}$ 设 $\hat\theta(X_1,\cdots, X_n)$ 是 $\theta$ 的一个估计量。$\Theta$ 为 $\theta$ 的取值范围。如果对于任意 $\theta\in \Theta$，有
$$
E[\hat\theta(X_1,\cdots, X_n)]=\theta
$$
则称 $\hat\theta$ 是 $\theta$ 的无偏估计量。

注：该式子的意思在于，虽然 $\theta$ 的估计量因为采样的原因可能和真实值有偏差，但采样次数足够多以后 $\hat\theta\rightarrow\theta$，即无偏性意味着没有系统误差。 

【例】设 $X_1,\cdots, X_n$ 来自总体 $X$，且$E[X^j]=\mu_j$ 存在，则 $A_j=\frac{1}{n}\sum_{k=1}^nX_k^j$ 是 $\mu_j$ 的无偏估计。这是因为
$$
E[A_j]=E\left[\frac{1}{n}\sum_{k=1}^nX_k^j\right]=\frac{1}{n}\sum_{k=1}^nE[X_k^j]=\frac{1}{n}\cdot nE[X^j]=E[X^j]
$$
【例】设 $X_1,\cdots, X_n$ 来自总体 $X$，且 $D(X)=\sigma^2$ 存在，则样本方差 $S^2$ 是 $\sigma^2$ 的无偏估计。这是因为
$$
\begin{align}
S^2&=\frac{1}{n-1}\sum_{k=1}^n(X_k-\overline{X})^2\\\\
&=\frac{1}{n-1}\sum_{k=1}^n(X_k^2-2X_k\overline{X}+\overline{X}^2)\\\\
&=\frac{1}{n-1}\left(\sum_{k=1}^nX_k^2-2\overline{X}\sum_{k=1}^nX_k+\sum_{k=1}^n\overline{X}^2\right)\\\\
&=\frac{1}{n-1}\left(\sum_{k=1}^nX_k^2-2n\overline{X}^2+n\overline{X}^2\right)\\\\
&=\frac{1}{n-1}\sum_{k=1}^nX_k^2+\frac{n}{n-1}\overline{X}^2
\end{align}
$$
从而
$$
\begin{align}
E[S^2]&=\frac{1}{n-1}\sum_{k=1}^nE[X_k^2]-\frac{n}{n-1}E[\overline{X}^2]\\\\
&=\frac{1}{n-1}\sum_{k=1}^n(\sigma^2+\mu^2)-\frac{n}{n-1}E\left[\left(\frac{1}{n}\sum_{k=1}^nX_k\right)^2\right]\\\\
&=\frac{n}{n-1}(\sigma^2+\mu^2)-\frac{1}{n(n-1)}\left(\sum_{k=1}^nE[X_k^2]+2\sum_{1\leq i<j\leq n}E[X_iX_j]\right)\\\\
&=\frac{n}{n-1}(\sigma^2+\mu^2)-\frac{1}{n(n-1)}\cdot n(\sigma^2+\mu^2)-\frac{2}{n(n-1)}\binom{n}{2}E[X]^2\\\\
&=\sigma^2+\mu^2-\mu^2\\\\
&=\sigma^2
\end{align}
$$
$E[S^{*2}]=E[\frac{n-1}{n}S^2]=\frac{n-1}{n}\sigma^2$，因此二阶中心矩不是方差的无偏估计。不过 $\lim_{n\rightarrow \infty}E[S^{*2}]=\sigma^2$，所以称 $S^{*2}$ 是 $\sigma^2$ 的渐进无偏估计。

【例题】设 $X_1,\cdots, X_n$ 来自总体 $X\sim U[0,\theta]$，令 $M_n=\max_{1\leq k\leq n}X_k$，求证：$\hat\theta=\frac{n+1}{n}M_n$ 是 $\theta$ 的无偏估计。

> 证明：$X$ 的分布函数为
> $$
> F(x)=\begin{cases}
> 0&,x\leq 0\\\\
> \frac{x}{\theta}&,0\leq x\leq \theta\\\\
> 1&,x>\theta
> \end{cases}
> $$
> 从而 $M_n$ 的分布函数为
> $$
> F_n(x)=\begin{cases}
> 0&,x\leq 0\\\\
> \left(\frac{x}{\theta}\right)^n&,0\leq x\leq \theta\\\\
> 1&,x>\theta
> \end{cases}
> $$
> 密度函数：
> $$
> p_n(x)=\begin{cases}
> \frac{n}{\theta}\left(\frac{x}{\theta}\right)^{n-1}&,0\leq x\leq \theta\\\\
> 0&,otherwise
> \end{cases}
> $$
> 从而
> $$
> E[\hat\theta]=E\left[\frac{n+1}{n}M_n\right]=\frac{n+1}{n}\int_0^\theta x\cdot \frac{n}{\theta}\left(\frac{x}{\theta}\right)^{n-1}dx=\theta
> $$

注：$2\overline{X}$ 也是 $\theta$ 的无偏估计，这说明无偏估计可能是不唯一的。

即是 $\hat\theta$ 是 $\theta$ 的无偏估计，$g(\hat\theta)$ 也未必是 $g(\theta)$ 的无偏估计，下面是一个例子：

【例】设 $\hat\theta$ 是 $\theta$ 的无偏估计，即 $E[\hat\theta]=\theta$，且 $D(\hat\theta)>0$，那么我们发现
$$
E[\hat\theta^2]=D(\hat\theta)+E[\hat\theta]^2=D(\hat\theta)+\theta^2\geq \theta^2
$$
因而 $(\hat\theta)^2$ 不是 $\theta^2$ 的无偏估计。

## Mean Square Error Criterion

$\fbox{Definition 20.2}$ 记 $M(\hat\theta,\theta)\triangleq E[(\hat\theta-\theta)^2]$ 为均方误差。

注：(1) $\hat\theta$ 是随机变量，我们希望对于任意 $\theta$，$M(\hat\theta,\theta)$ 尽量小。

(2) 
$$
\begin{align}
M(\hat\theta,\theta)&=E[(\hat\theta-\theta)^2]=E[(\hat\theta-E[\hat\theta]+E[\hat\theta]-\theta)^2]\\\\
&=E[(\hat\theta-E[\hat\theta])^2]+2E[(\hat\theta-E[\hat\theta])(E[\hat\theta]-\theta)]+E[(E[\hat\theta]-\theta)^2]\\\\
&=D(\hat\theta)+(E[\hat\theta]-\theta)^2
\end{align}
$$
(注：$\hat\theta$ 是一个随机变量，$\theta$ 是一个数。)

当 $\hat\theta$ 是 $\theta$ 的无偏估计时，$M(\hat\theta,\theta)=D(\hat\theta)$。

【例】设 $X_1,\cdots, X_n$ 来自总体 $X\sim U[0,\theta]$，现比较 $\hat{\theta_0}=2X_1$，$\hat{\theta_1}=2\overline{X}$ 和 $\hat{\theta_2}=\frac{n+1}{n}M_n$ 的均方误差。注意到这三者都是无偏估计，从而
$$
\begin{align}
M(\hat\theta_0,\theta)&=D(\hat\theta_0)=D(2X_1)=4D(X)=\frac{4\theta^2}{12}=\frac{\theta^2}{3}\\\\
M(\hat\theta_1,\theta)&=D(\hat\theta_1)=D(2\overline{X})=4D\left(\frac{1}{n}\sum_{k=1}^nX_k\right)=\frac{4}{n^2}\sum_{k=1}^nD(X_k)=\frac{4}{n}\cdot\frac{\theta^2}{12}=\frac{\theta^2}{3n}\\\\
M(\hat\theta_2,\theta)&=D(\hat\theta_2)=\left(\frac{n+1}{n}\right)^2D(M_n)=\left(\frac{n+1}{n}\right)^2(E[M_n^2]-E[M_n]^2)\\\\
&=\left(\frac{n+1}{n}\right)^2\left(\int_0^\theta x^2\frac{n}{\theta}\left(\frac{x}{\theta}\right)^{n-1}dx-\left(\frac{n}{n+1}\theta\right)^2\right)\\\\
&=\left(\frac{n+1}{n}\right)^2\left(\frac{n}{n+2}\theta^2-\left(\frac{n}{n+1}\right)^2\theta^2\right)\\\\
&=\frac{1}{n(n+2)}\theta^2
\end{align}
$$
注：(1) 对比 $\hat\theta_1$ 和 $\hat\theta_0$ 可知，增加采样数可以提高估计量的精度。

(2) 注意到 $M_n$ 是极大似然估计，类似地可算得 $M(M_n,\theta)\sim\frac{\theta^2}{n^2}$。而 $2\overline{X}$ 是矩估计，故在本例中极大似然估计优于矩估计。

## Consistency

$\fbox{Definitino 20.3}$ 设 $\hat\theta_n(X_1,\cdots, X_n)$ 是 $\theta$ 的一个估计量，若对于任意 $\theta\in \Theta$，有 $\hat\theta_n\overset{P}{\rightarrow}\theta$，则称 $\hat\theta_n$ 是 $\theta$ 的一致估计量。

注：(1) 矩估计具有一致性，在一定条件下可以证明极大似然估计也具有一致性。

(2) 不具有一致性 (增加采样数不能提高精度) 的估计量不可取。

【例】设 $X_1,\cdots, X_n$ 来自总体 $X\sim U[0,\theta]$，则 $\hat\theta_1=2\overline{X}$，$\hat\theta_2=\frac{n+1}{n}M_n$  和 $\hat\theta_3=M_n$ 都具有一致性。这是因为

(1) $\hat\theta_1=\frac{2}{n}\sum_{k=1}^nX_k$，$X_1,\cdots,X_n$ 独立同分布且方差一致有界，所以 $\hat\theta_1$ 服从大数定律，即对于任意 $\varepsilon>0$，
$$
\begin{align}
&\quad\space\lim_{n\rightarrow\infty}P\left(\left|\hat\theta_1-\frac{2}{n}\sum_{k=1}^nE[X_k]\right|>\varepsilon\right)=0\\\\
&\Rightarrow\lim_{n\rightarrow\infty}P\left(\left|\hat\theta_1-\frac{2}{n}\cdot n\frac{\theta}{2}\right|>\varepsilon\right)=0\\\\
&\Rightarrow\lim_{n\rightarrow\infty}P\left(\left|\hat\theta_1-\theta\right|>\varepsilon\right)=0
\end{align}
$$
所以 $\hat\theta_1\overset{P}{\rightarrow}\theta$。

(2) $E[\hat\theta_2]=\theta$，$D(\hat\theta_2)=\frac{1}{n(n+2)}\theta^2$，由切比雪夫不等式得
$$
P\left(\left|\hat\theta_2-\theta\right|>\varepsilon\right)=P\left(\left|\hat\theta_2-E[\hat\theta_2]\right|>\varepsilon\right)\leq\frac{1}{\varepsilon^2}D(\hat\theta_2)=\frac{\theta^2}{\varepsilon^2}\frac{1}{n(n+2)}\overset{n\rightarrow \infty}{\longrightarrow}0
$$
(3) $E[\hat\theta_3]=\frac{n}{n+1}\theta$，$D(\hat\theta_3)=\frac{n}{(n+1)^2(n+2)}\theta^2$。
$$
\begin{align}
P\left(\left|\hat\theta_3-\theta\right|>\varepsilon\right)&=P\left(\left|\hat\theta_3-\frac{n}{n+1}\theta-\frac{1}{n+1}\theta\right|>\varepsilon\right)\\\\
&\overset{triangle}{\leq} P\left(\left|\hat\theta_3-\frac{n}{n+1}\theta\right|+\left|\frac{1}{n+1}\theta\right|>\varepsilon\right)\\\\
&\leq P\left(\left\{\left|\hat\theta_3-\frac{n}{n+1}\theta\right|>\frac{\varepsilon}{2}\right\}\cup\left\{\frac{1}{n+1}\theta>\frac{\varepsilon}{2}\right\}\right)\\\\
&\leq P\left(\left|\hat\theta_3-\frac{n}{n+1}\theta\right|>\frac{\varepsilon}{2}\right)+P\left(\frac{1}{n+1}\theta>\frac{\varepsilon}{2}\right)
\end{align}
$$
当 $n\rightarrow \infty$ 时，后一项趋于 0，前一项现在的形式是 $\hat\theta_3-E[\hat\theta_3]$ 的形式，因此可以运用切比雪夫不等式：
$$
P\left(\left|\hat\theta_3-\frac{n}{n+1}\theta\right|>\frac{\varepsilon}{2}\right)\leq \frac{4}{\varepsilon^2}D(\hat\theta_3)=\frac{4n}{(n+1)^2(n+2)}\theta^2\overset{n\rightarrow\infty}{\longrightarrow}0
$$

