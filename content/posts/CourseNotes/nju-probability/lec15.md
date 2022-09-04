---
title: "Lecture 15: Covariance and Correlation Coefficient"
linktitle: Lecture 15
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
        weight: 15

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
--- 

$\fbox{Theorem 15.1}$ (Chebyshev) 设随机变量 $X$ 的方差存在，则有
$$
\forall \varepsilon>0,P(|X-E[X]|>\varepsilon)\leq \frac{D(X)}{\varepsilon^2}
$$
证明：设 $X$ 存在密度函数 $p(x)$，则
$$
\begin{align}
P(|X-E[X]|>\varepsilon)&=\int_{\{x:|X-E[X]|>\varepsilon\}}p(x)dx\\\\
&\leq \int_{\{x:|X-E[X]|>\varepsilon\}}\frac{|X-E[x]|^2}{\varepsilon^2}p(x)dx\\\\
&\leq \frac{1}{\varepsilon^2}\int_{-\infty}^{+\infty}|X-E[X]|^2p(x)dx\\\\
&=\frac{D(X)}{\varepsilon^2}\qquad \blacksquare
\end{align}
$$
我们的证明过程中使用了两次看上去很“松”的放缩，但下面的例子可以说明，在对 $X$ 没有任何额外限制的情况下，Chebyshev 的结果已经是最紧的了：

例：设 $X$ 的分布律为 $P(X=1)=P(X=-1)=\frac{1}{2k^2}$，$P(X=0)=1-\frac{1}{k^2}$，显然有 $E[X]=0$。那么
$$
P(|X-E[x]|\geq 1)=P(X=\pm 1)=\frac{1}{k^2}\leq \frac{D(X)}{\varepsilon^2}=\frac{1/k^2}{1^2}=\frac{1}{k^2}
$$
【例题】用 Chebyshev 不等式证明：如果随机变量 $X$ 满足 $D(X)=0$，则 $P(X=E[X])=1$。

> 证明：令 $A=\{w:|X(w)-E[X]|>0\}$，我们要证明的目标是 $P(A)=0$。
>
> 令 $A_n=\{w:|X(w)-E[X]|>\frac{1}{n}\}$，那么显然有 $A=\bigcup_{n=1}^\infty A_n$，从而
> $$
> P(A)=P(\bigcup_{n=1}^\infty A_n)\leq \sum_{n=1}^\infty P(A_n)=\sum_{n=1}^\infty P(X-E[X]>\frac{1}{n})\leq \sum_{n=1}^\infty \frac{1}{n^2}D(X)=0
> $$
> 又 $P(A)\geq 0$，所以 $P(A)=0$。
>
> （注：第一个不等号是 union bound。注意 $A_n$ 之间不是互不相交的，所以不能用“可列可加性”。）

## Covariance

$\fbox{Definition 15.2}$ $X,Y$ 为随机变量，$E[X],E[Y],E[XY]$ 均存在，则
$$
cov(X,Y)\triangleq E[(X-E[X])(Y-E[Y])]
$$
为 $X$ 和 $Y$ 的**协方差**。

注：(1) $cov(X,X)=D(X)$。

(2) $D(X\pm Y)=D(X)+D(Y)\pm 2cov(X,Y)$。

一般地，
$$
D(\sum_{k=1}^nX_k)=\sum_{k=1}^nD(X_k)+2\sum_{1\leq i<j\leq n}cov(X_i,X_j).
$$
(3) $cov(X,Y)=E[XY-XE[Y]-YE[X]+E[X]E[Y]]=E[XY]-E[X]E[Y]$。

(4) $cov(X,Y)>0$ 说明 $X$ 和 $Y$ 有很大倾向同时大于或小于各自的期望；$cov(X,Y)<0$ 说明 $X$ 和 $Y$ 有很大倾向一个大于自身期望，一个小于自身期望。

(5) $cov(X,Y)$ 依赖于单位。比如如果将两者的单位本来是米，将它们改成毫米，协方差的数值将会变大很多。因此协方差不是一个很好的衡量相关性的客观数值。

协方差的性质：

* $cov(X,Y)=cov(Y,X)$。

* 对于任意 $a,b,c,d\in \mathbb R$，$cov(aX+c,bY+d)=abcov(X,Y)$。

    > 证明：
    > $$
    > \begin{align}
    > cov(aX+c,bY+d)&=E[(aX+c)(bY+d)]-E[aX+c]E[bY+d]\\\\
    > &=E[abXY+adX+bcY+cd]-(aE[X]+c)(bE[Y]+d)\\\\
    > &=abE[XY]-abE[X]E[Y]\\\\
    > &=abcov(X,Y)
    > \end{align}
    > $$

* $cov(X_1+X_2,Y)=cov(X_1,Y)+cov(X_2,Y)$。

    > 证明：
    > $$
    > \begin{align}
    > cov(X_1+X_2,Y)&=E[(X_1+X_2)Y]-E[X_1+X_2]E[Y]\\\\
    > &=E[X_1Y]+E[X_2Y]-E[X_1]E[Y]-E[X_2]E[Y]\\\\
    > &=cov(X_1,Y)+cov(X_2,Y)
    > \end{align}
    > $$

* 若 $X,Y$ 独立，则 $cov(X,Y)=0$。

    > 证明：$cov(X,Y)=E[XY]-E[X]E[Y]=E[X]E[Y]-E[X]E[Y]=0$。

下面展示一个比较精巧的计算协方差的例子：

【例】设 $(X,Y)$ 服从参数为 $n,p_1,p_2$ 的三项分布，即对于任意 $i,j$，$i+j\leq n$，
$$
P(X=i,Y=j)=\frac{n!}{i!j!(n-i-j)!}p_1^ip_2^j(1-p_1-p_2)^{n-i-j}
$$
考虑计算 $cov(X,Y)$。我们发现利用定义计算非常麻烦，但考虑公式
$$
D(X+Y)=D(X)+D(Y)+2cov(X,Y)
$$
这三个方差都很容易求：因为 $X\sim B(n,p_1),Y\sim B(n,p_2),X+Y\sim B(n,p_1+p_2)$，所以
$$
\begin{align}
cov(X,Y)&=\frac{1}{2}(D(X+Y)-D(X)-D(Y))\\\\
&=\frac{1}{2}(n(p_1+p_2)(1-p_1-p_2)-np_1(1-p_2)-np_2(1-p_2))\\\\
&=-np_1p_2
\end{align}
$$
$\fbox{Theorem 15.3}$ (Cauchy-Schwarz) 设随机变量 $X,Y$ 的方差都存在，则
$$
(cov(X,Y))^2\leq D(X)D(Y)
$$
其中等号取到当且仅当存在不全为 0 的常数 $a,b$，使得 $P(Y=aX+b)=1$。

证明：对于任意 $t\in \mathbb R$，令
$$
u(t)=E[(t(X-E[X])-(Y-E[Y]))^2]\geq 0
$$
那么
$$
\begin{align}
u(t)&=t^2E[(X-E[X])^2]-2tE[(X-E[X])(Y-E[Y])]+E[(Y-E[Y])^2]\\\\
&=t^2D(X)-2tcov(X,Y)+D(Y)
\end{align}
$$
因为 $u(t)\geq 0$，所以二次函数判别式小于等于 0，即 $(cov(X,Y))^2\leq D(X)D(Y)$。

下面考虑等号成立的条件：

$\Rightarrow:$ 若 $(cov(X,Y))^2=D(X)D(Y)$，那么存在 $t_0\in \mathbb R$，$u(t_0)=0$。令 $Z=t_0(X-E[X])-(Y-E[Y])$，则
$$
0=u(t_0)=E[Z^2]\geq D(Z)\geq 0
$$
所以 $D(Z)=0$，$P(Z=E[Z])=P(Z=0)=1$。

$\Leftarrow:$ 若存在不全为 0 的 $a,b\in \mathbb R$ 满足 $P(Y=aX+b)=1$，那么
$$
\begin{align}
cov(X,Y)&=E[XY]-E[X]E[Y]\\\\
&=E[X(aX+b)]-E[X]E[aX+b]\\\\
&=aE[X^2]+bE[X]-a(E[X])^2-bE[X]\\\\
&=aD(X)=\pm \sqrt{a^2D(X)^2}
\end{align}
$$
注意到 $D(Y)=D(aX+b)=a^2D(X)$，所以 $cov(X,Y)=\pm \sqrt{D(X)D(Y)}$。$\blacksquare$

## Correlation Coefficient

由 Cauchy-Schwarz 不等式可知，$D(X),D(Y)>0$ 时，$\left|\frac{cov(X,Y)}{\sqrt{D(X)D(Y)}}\right|\leq 1$，我们在此基础上定义相关系数。

$\fbox{Definition 15.4}$ 设随机变量 $X,Y$ 方差均存在，$D(X)>0,D(Y)>0$，则令
$$
\rho_{XY}\triangleq \frac{cov(X,Y)}{\sqrt{D(X)D(Y)}}
$$
为 $X,Y$ 的**相关系数**，也可记作 $corr(X,Y)$。若 $\rho_{XY}>0$，则称 $X,Y$ 正相关；若 $\rho_{XY}<0$，则称 $X,Y$ 负相关。

注：(1) 考虑 $X,Y$ 的标准化随机变量 $X^\*=\frac{X-E[X]}{\sqrt{D(X)}},Y^\*=\frac{Y-E[Y]}{\sqrt{D(Y)}}$，那么
$$
cov(X^\*,Y^\*)=E[(X^\*-E[X^\*])(Y^\*-E[Y^\*])]=E\left[\frac{X-E[X]}{\sqrt{D(X)}}\cdot \frac{Y-E[Y]}{\sqrt{D(Y)}}\right]=\frac{cov(X,Y)}{\sqrt{D(X)D(Y)}}=\rho_{XY}
$$
即标准化后的随机变量的协方差等于其原变量的相关系数。

(2) $\rho_{XY}$ 不依赖 $X,Y$ 数量级的选取。

(3) 根据 Cauchy-Schwarz 的取等条件，$|\rho_{XY}=1|$ 当且仅当 $|cov(X,Y)|=\sqrt{D(X)D(Y)}$，即存在不全为 0 的常数 $a,b$，$P(Y=aX+b)=1$。$\rho_{XY}$ 刻画了 $X,Y$ 之间线性相关的程度，$|\rho_{XY}|$ 越大，$X,Y$ 的线性关系就越密切。

$\fbox{Definition 15.5}$ 若 $X,Y$ 满足 $\rho_{XY}=0$，则称 $X,Y$ **线性无关**或**不相关**。

$\fbox{Proposition 15.6}$ 在 $X,Y$ 方差存在的情况下，下述 4 条等价：

* $\rho_{XY}=0$。
* $cov(X,Y)=0$。
* $E\[XY\]=E\[X\]E\[Y\]$。
* $D(X\pm Y)=D(X)+D(Y)$。

根据上述命题，我们容易发现如果 $X,Y$ 相互独立，那么 $\rho_{XY}=0$，但反之不成立，即不相关是比相互独立弱的条件，下面是一个例子：

【例】设 $\theta$ 服从 $[0,2\pi]$ 上的均匀分布，令 $X=\cos \theta,Y=\cos (\theta+a)$，其中 $a$ 为给定常数，考虑 $\rho_{XY}$。
$$
\begin{align}
E[X]&=\int_0^{2\pi}xp(x)dx=\frac{1}{2\pi}\int_0^{2\pi}\cos xdx=0\\\\
E[Y]&=\frac{1}{2\pi}\int_0^{2\pi}\cos(x+a)dx=0\\\\
D(X)&=E[(X-0)^2]=\frac{1}{2\pi}\int_0^{2\pi}\cos ^2xdx=\frac{1}{4\pi}\int_0^{2\pi}(1+\cos 2x)dx=\frac{1}{2}\\\\
D(Y)&=E[(Y-0)^2]=\frac{1}{2\pi}\int_0^{2\pi}\cos ^2(x+a)dx=\frac{1}{4\pi}\int_0^{2\pi}(1+\cos 2(x+a))dx=\frac{1}{2}\\\\
E[XY]&=E[\cos \theta \cos (\theta +a)]=\frac{1}{2\pi}\int_0^{2\pi}\cos x\cos (x+a)dx\\\\
&\overset{和差化积}{=}\frac{1}{4\pi}(\int_0^{2\pi}\cos(2x+a)+\cos a)dx=\frac{1}{2}\cos a\\\\
\rho_{XY}&=\frac{cov(X,Y)}{\sqrt{D(X)D(Y)}}=\frac{\frac{1}{2}cos a}{\sqrt{\frac{1}{2}\cdot \frac{1}{2}}}=\cos a
\end{align}
$$
当 $a=\frac{3\pi}{2}$ 时，$\rho_{XY}=0$，$X,Y$ 不相关。此时 $X=\cos \theta,Y=\cos (\theta+a)=\sin \theta$，有 $X^2+Y^2=1$。

考虑如下事件的概率：$P(|X|\leq \frac{1}{2},|Y|\leq \frac{1}{2})$，因为 $X^2+Y^2=1$，所以该概率为 0。但 $P(|X|\leq \frac{1}{2})>0$，$P(|Y|\leq \frac{1}{2})>0$，从而 $P(|X|\leq \frac{1}{2},|Y|\leq \frac{1}{2})\neq P(|X|\leq \frac{1}{2})\cdot P(|Y|\leq \frac{1}{2})$，所以 $X,Y$ 不相互独立。

【例题】设 $(X,Y)\sim N(\mu_1,\mu_2,\sigma_1^2,\sigma_2^2,\rho)$，求 $cov(X,Y)$ 和 $\rho_{XY}$。

解：
$$
p(x,y)=\frac{1}{2\pi\sigma_1\sigma_2\sqrt{1-\rho^2}}e^{-\frac{1}{2(1-\rho^2)}\left[\left(\frac{x-\mu_1}{\sigma_1}\right)^2-2\rho \left(\frac{x-\mu_1}{\sigma_1}\cdot \frac{y-\mu_2}{\sigma_2}\right)+\left(\frac{y-\mu_2}{\sigma_2}\right)^2\right]}
$$
我们已知 $E[X]=\mu_1,E[Y]=\mu_2,D(X)=\sigma_1^2,D(Y)=\sigma_2^2$。
$$
\begin{align}
cov(X,Y)&=E[(X-E[X])(Y-E[Y])]=\int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty}(x-\mu_1)(y-\mu_2)p(x,y)dxdy\\\\
&\overset{u=\frac{x-\mu_1}{\sigma_1},v=\frac{y-\mu_2}{\sigma_2}}{=}\frac{1}{2\pi\sigma_1\sigma_2\sqrt{1-\rho^2}}\int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty}\sigma_1u\cdot \sigma_2ve^{-\frac{1}{2(1-\rho^2)}(u^2-2\rho uv+v^2)}d(\sigma_1u)d(\sigma_2v)\\\\
&=\frac{\sigma_1\sigma_2}{2\pi\sqrt{1-\rho^2}}\int_{-\infty}^{+\infty}v\int_{-\infty}^{+\infty}ue^{-\frac{1}{2(1-\rho^2)}(u^2-2\rho vu+\rho^2v^2-\rho^2v^2+v^2)}dudv\\\\
&=\frac{\sigma_1\sigma_2}{2\pi\sqrt{1-\rho^2}}\int_{-\infty}^{+\infty}ve^{-\frac{v^2}{2}}\int_{-\infty}^{+\infty}ue^{-\frac{1}{2(1-\rho^2)}(u-\rho v)^2}dudv
\end{align}
$$
注意到
$$
\frac{1}{\sqrt{2\pi}\sqrt{1-\rho^2}}\int_{-\infty}^{+\infty}ue^{-\frac{1}{2(1-\rho^2)}(u-\rho v)^2}du
$$
是正态分布 $N(\rho v, 1-\rho^2)$ 的期望 (自变量为 $u$)，所以
$$
\begin{align}
cov(X,Y)=\frac{\sigma_1\sigma_2}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}ve^{-\frac{v^2}{2}}\rho vdv=\sigma_1\sigma_2\rho\cdot  \frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}v^2e^{-\frac{v^2}{2}}dv
\end{align}
$$
注意到
$$
\frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}v^2e^{-\frac{v^2}{2}}dv=\frac{1}{\sqrt{2\pi}}\int_{-\infty}^{+\infty}(v-0)^2e^{-\frac{v^2}{2}}dv
$$
是标准正态分布 $N(0,1)$ 的方差，所以该式为 1，所以 $cov(X,Y)=\sigma_1\sigma_2\rho$。相关系数 $\rho_{XY}=\rho$。

$\fbox{Theorem 15.7}$ 设 $(X,Y)\sim N(\mu_1,\mu_2,\sigma_1^2,\sigma_2^2,\rho)$，则 $X,Y$ 相互独立当且仅当 $X,Y$ 不相关。

