---
title: " Lecture 16: Moment and Covariance Matrix"
linktitle: Lecture 16
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
        weight: 16

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
--- 

## Origin Moment and Central Moment

$\fbox{Definition 16.1}$ 设 $X$ 是随机变量，对正整数 $k$，若 $E[|X|^k]<+\infty$，则称 $E[X^k]$ 为 $X$ 的 $k$ 阶原点矩或 $k$ 阶矩；若 $E[|X-E[X]|^k]<+\infty$，则称 $E[(X-E[X])^k]$ 为 $X$ 的 $k$ 阶中心矩。

注：(1) 矩是对期望、方差的推广，1 阶原点矩就是期望，2 阶中心矩就是方差。

(2) 一般情形下，两个随机变量的任意阶原点矩/中心矩相同不能推出两个随机变量的分布相同。

【例】考虑正态分布 $X\sim N(\mu,\sigma^2)$ 的 $k$ 阶中心矩。首先写出 $X$ 的密度函数：
$$
p(x)=\frac{1}{\sqrt{2\pi}\sigma}e^{-\frac{(x-\mu)^2}{2\sigma^2}}
$$
所以
$$
\begin{align}
E[(X-E[X])^k]&=\frac{1}{\sqrt{2\pi}\sigma}\int_{-\infty}^{+\infty}(x-\mu)^ke^{-\frac{(x-\mu)^2}{2\sigma^2}}dx\\\\
&\overset{t=\frac{x-\mu}{\sigma}}{=}\frac{1}{\sqrt{2\pi}\sigma}\int_{-\infty}^{+\infty}\sigma^kt^ke^{-\frac{t^2}{2}}d(\sigma t)\\\\
&=\frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}t^ke^{-\frac{t^2}{2}}dt
\end{align}
$$
$k$ 是奇数时，被积函数是奇函数，原式为 0。$k$ 是偶数时，我们考虑归纳地使用分部积分：
$$
\begin{align}
\int\_{-\infty}^{+\infty}t^ke^{-\frac{t^2}{2}}dt&=\int\_{-\infty}^{+\infty}t^{k-1}e^{-\frac{t^2}{2}}d\frac{t^2}{2}=-\int\_{-\infty}^{+\infty}t^{k-1}de^{-\frac{t^2}{2}}\\\\
&=-\left.t^{k-1}e^{-\frac{t^2}{2}}\right|\_{-\infty}^{+\infty}+\int\_{-\infty}^{+\infty}e^{-\frac{t^2}{2}}d(t^{k-1})\\\\
&=(k-1)\int\_{-\infty}^{+\infty}t^{k-2}e^{-\frac{t^2}{2}}dt
\end{align}
$$
最后剩下的东西是高斯积分，为 $\sqrt{2\pi}$，因此
$$
E[(X-E[X])^k]=\begin{cases}0&,k是偶数\\\\\sigma^k(k-1)!!&,k是奇数\end{cases}.
$$
注：标准化随机变量 (此时期望为 0，中心矩和原点矩相同) 的三阶矩称为偏度。偏度为负意味着分布的左尾更长，但大多数取值比期望大 (即有一些概率取到很小的负数，但更大概率取到大于期望的数)；偏度为 0 意味着数值相对均匀地分布在期望的两侧 (但未必完全对称)。

四阶矩称为峰度。峰度高意味着存在远大于或远小于期望的取值且取到的概率不太小。

对于中心矩，我们有如下的切比雪夫不等式的推广：

$\fbox{Theorem 16.2}$ 设 $E[|X-E[X]|^k]<+\infty$，则对于任意 $\varepsilon>0$，
$$
P(|X-E[X||>\varepsilon)\leq \frac{E[|X-E[X]|^k]}{\varepsilon^k}
$$
该不等式在远端可以给出比切比雪夫不等式更好的估计。

## Covariance Matrix

$\fbox{Definition 16.3}$ 设 $X=(X_1,\cdots, X_n)^T$ 是一个 $n$ 维随机变量，则
$$
E[X]\triangleq(E[X_1],E[X_2],\cdots, E[X_n])^T
$$
为 $X$ 的期望。记 $C_{ij}=cov(X_i,X_j)$，称矩阵
$$
\Sigma=\begin{bmatrix}C_{11} & C_{12}&\cdots &C_{1n}\\\\
C_{21}&C_{22}&\cdots&C_{2n}\\\\
\vdots&\vdots&\ddots&\vdots\\\\
C_{n1}&C_{n2}&\cdots&C_{nn}\end{bmatrix}
$$
为 $X$ 的协方差阵。

【例】考虑 $(X_1,X_2)\sim N(\mu_1,\mu_2,\sigma_1^2,\sigma_2^2,\rho)$，记 $\Sigma=\begin{bmatrix}\sigma_1^2& \sigma_1\sigma_2\rho\\\\\sigma_1\sigma_2\rho&\sigma_2^2\end{bmatrix}$，$\mu=(\mu_1,\mu_2)^T,X=(X_1,X_2)^T$，则
$$
p(x_1,x_2)=p(x)=\frac{1}{2\pi\sqrt{\det \Sigma}}e^{-\frac{1}{2}(X-\mu)^T\Sigma^{-1}(X-\mu)}
$$
$\fbox{Theorem 16.4}$ 协方差矩阵是半正定的，即对于任意 $X\in \mathbb R^n$，$X^T\Sigma X\geq 0$。

证明：对于任意 $T=(t_1,\cdots, t_n)^T\in \mathbb R^n$，考虑如下随机变量的期望：
$$
E\left[\left(\sum_{i=1}^nt_i(X_i-E[X_i])\right)^2\right]\geq 0
$$
我们下面证明这个期望就是协方差矩阵的二次型：
$$
\begin{align}
E\left[\left(\sum_{i=1}^nt_i(X_i-E[X_i])\right)^2\right]&=E\left[\sum_{i=1}^n\sum_{j=1}^nt_it_j(X_i-E[X_i])(X_j-E[X_j])\right]\\\\
&=\sum_{i=1}^n\sum_{j=1}^nt_it_jE[(X_i-E[X_i])(X_j-E[X_j])]\\\\
&=\sum_{i=1}^n\sum_{j=1}^nt_it_jC_{ij}\\\\
&=\sum_{j=1}^n\left(\sum_{i=1}^nt_iC_{ij}\right)t_j\\\\
&=
\begin{pmatrix}
\sum_{i=1}^nt_iC_{i1}&\sum_{i=1}^nt_iC_{i2}&\cdots&\sum_{i=1}^nt_iC_{in}
\end{pmatrix}
\begin{pmatrix}
t_1\\\\t_2\\\\\vdots\\\\t_n
\end{pmatrix}\\\\
&=\begin{pmatrix}
\begin{pmatrix}
t_1&\cdots&t_n
\end{pmatrix}
\begin{pmatrix}
C_{11}\\\\\vdots \\\\C_{1n}
\end{pmatrix}
&
\cdots
&
\begin{pmatrix}
t_1&\cdots&t_n
\end{pmatrix}
\begin{pmatrix}
C_{n1}\\\\\vdots \\\\C_{nn}
\end{pmatrix}
\end{pmatrix}
\begin{pmatrix}
t_1\\\\\vdots \\\\t_n
\end{pmatrix}\\\\
&=
\begin{pmatrix}
t_1&\cdots &t_n
\end{pmatrix}
\begin{bmatrix}C_{11} & C_{12}&\cdots &C_{1n}\\\\
C_{21}&C_{22}&\cdots&C_{2n}\\\\
\vdots&\vdots&\ddots&\vdots\\\\
C_{n1}&C_{n2}&\cdots&C_{nn}\end{bmatrix}
\begin{pmatrix}
t_1\\\\\vdots \\\\t_n
\end{pmatrix}\\\\
&=T^T\Sigma T
\end{align}
$$
所以 $\Sigma$ 半正定。 $\blacksquare$

