---
title: "Lecture 09: 2-Dimensional Continuous Random Variable"
linktitle: Lecture 09
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
        weight: 9

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
--- 

## Examples of Discrete 2-D Random Variable

【例】 (三项分布) 若二维离散型随机变量 $(X,Y)$ 的分布律为
$$
P(X=i,Y=j)=\frac{n!}{i!j!(n-i-j)!}p_1^ip_2^j(1-p_1-p_2)^{n-i-j}
$$
其中 $i,j=0,1,\cdots,n,i+j\leq n,0\leq p_1,p_2,p_1+p_2\leq 1$，则称 $(X,Y)$ 服从参数 $n,p_1,p_2$ 的三项分布。

概率背景：在 $n$ 重独立重复试验中，每次试验有三种可能的结果 $A_1,A_2,A_3$，$P(A_1)=p_1,P(A_2)=p_2$。令 $A_1$ 发生次数为 $X$，$A_2$ 发生次数为 $Y$，则 $(X,Y)$ 服从上述三项分布。

三项分布的边缘分布是二项分布，因为在计算 $P(X=k)$ 时，我们不关心在没有命中 $A_1$ 时命中的是 $A_2$ 还是 $A_3$，相当于只剩下了两种事件。我们也可以从代数上进行验证：
$$
\begin{align}
P(X=k)&=P(X=k,0\leq Y\leq n-k)=\sum_{i=0}^{n-k}P(X=k,Y=i)\\\\
&=\sum_{i=0}^{n-k}\frac{n!}{k!i!(n-k-i)!}p_1^kp_2^i(1-p_1-p_2)^{n-k-i}\\\\
&=\frac{n!}{k!(n-k)!}p_1^k\sum_{i=0}^{n-k}\frac{(n-k)!}{i!(n-k-i)!}p_2^i(1-p_1-p_2)^{n-k-i}\\\\
&=\frac{n!}{k!(n-k)!}p_1^k(p_2+(1-p_1-p_2))^{n-k}\\\\
&=\binom{n}{k}p_1^k(1-p_1)^{n-k}
\end{align}
$$
一般地，可以定义 $k$ 项分布。记 $P(A_1)=p_1,\cdots, P(A_k)=p_k$，$0\leq p_1,\cdots, p_k,\sum_{i=1}^kp_i\leq 1$，记 $X_j$是 $n$ 次试验中 $A_j$ 发生的次数，则 $(X_1,\cdots, X_k)$ 的分布律为
$$
P(X_1=j_1,\cdots, X_k=j_k)=\frac{n!}{j_1!\cdots j_k!}p_1^{j_1}\cdots p_k^{j_k}
$$
其中 $0\leq j_1,\cdots, j_k\leq n,\sum_{i=1}^kj_i=n$。

【例】 (二维超几何分布) 若二维离散型随机变量 $(X,Y)$ 的分布律为 
$$
P(X=n_1,Y=n_2)=\frac{\binom{N_1}{n_1}\binom{N_2}{n_2}\binom{N_3}{n_3}}{\binom{N}{n}}
$$
其中 $0\leq n_1\leq N_1,0\leq n_2\leq N_2,0\leq n_3\leq N_4,n_1+n_2+n_3=n,N_1+N_2+N_3=N$，则称 $(X,Y)$ 服从二维超几何分布。

概率背景：设 $N$ 个物品分为三类，各有 $N_1,N_2,N_3$ 个，不放回地挑 $n$ 个，第一类抽到 $X$ 个，第二类抽到 $Y$ 个，则 $(X,Y)$ 服从上述二维超几何分布。

类似地，二维超几何分布的边缘分布是一维的超几何分布：
$$
\begin{align}
P(X=n_1)&=P(X=n_1,0\leq Y\leq \min\{N_2,n-n_1\})\\\\
&=\sum_{k=0}^{\min\{N_2,n-n_1\}}\frac{\binom{N_1}{n_1}\binom{N_2}{k}\binom{N_3}{n-n_1-k}}{\binom{N}{n}}\\\\
&=\frac{\binom{N_1}{n_1}}{\binom{N}{n}}\left(\sum_{k=0}^{\min\{N_2,n-n_1\}}\binom{N_2}{k}\binom{N_3}{n-n_1-k}\right)\\\\
&=\frac{\binom{N_1}{n_1}}{\binom{N}{n}}\binom{N_2+N_3}{n-n_1}\\\\
&=\frac{\binom{N_1}{n_1}\binom{N-N_1}{n-n_1}}{\binom{N}{n}}\qquad (一维超几何分布)
\end{align}
$$

## 2-Dimensional Continuous Random Variable

$\fbox{Definition 9.1}$ 对于一个二维随机变量 $(X,Y)$ 的分布函数 $F(x,y)$，若存在非负可积函数 $p(x,y)$，使得对于任意 $(x,y)\in \mathbb{R}^2$，有
$$
F(x,y)=\int_{-\infty}^x\int_{-\infty}^yp(u,v)dudv
$$
则称 $(X,Y)$ 是**二维连续型随机变量**，称 $p(x,y)$ 是 $(X,Y)$ 的 (联合) 概率密度函数。

$p(x,y)$ 的性质：

* $\forall (x,y)\in \mathbb{R}^2,p(x,y)\geq 0$

* $$
    \int_{-\infty}^{+\infty}\int_{-\infty}^{+\infty}p(x,y)dxdy=F(+\infty,+\infty)=1
    $$

* 设 $D\subseteq \mathbb{R}^2$，则 $(X,Y)$ 落入 $D$ 中的概率为
    $$
    P((X,Y)\in D)=\iint_D p(x,y)dxdy
    $$

* 若 $p(x,y)$ 在 $(x_0,y_0)$ 附近连续，则有
    $$
    \left.\frac{\partial^2 F(x,y)}{\partial x\partial y}\right|\_{(x,y)=(x_0,y_0)}=p(x_0,y_0)
    $$
    注：和一维的情形类似，在二维连续型随机变量中，$p(x_0,y_0)$ 不能理解为 $x=x_0,y=y_0$ 的概率。事实上，$\forall x_0,y_0\in \mathbb{R},P(X=x_0,Y=y_0)=0$。$p(x_0,y_0)$ 只能理解为 $(X,Y)$ 落入 $(x_0,y_0)$ 附近一小块面积的概率的近似值，即
    $$
    \int_{x_0}^{x_0+\Delta x}\int_{y_0}^{y_0+\Delta y}p(x,y)dxdy\approx p(x_0,y_0)\Delta x\Delta y
    $$

边缘分布和密度函数的求法：

$X$ 的边缘分布函数为
$$
F_X(x)=F(X,+\infty)=\int_{-\infty}^x\left[\int_{-\infty}^{+\infty}p(u,y)dy\right]du
$$
$X$ 的边缘密度函数为
$$
p_X(x)=\frac{d}{dx}F_X(x)=\int_{-\infty}^{\infty}p(x,y)dy
$$
(直观地想，离散时边缘密度的求法是固定 $x$，对所有可能的 $y$ 求和，那么在连续型中将求和换作积分即可。)

$Y$ 的边缘分布和密度函数求法类似。

关于独立性：对于一般的二维随机变量，$X,Y$ 的独立性定义为
$$
\forall (x,y)\in \mathbb{R}^2,F(x,y)=F_X(x)F_Y(y)
$$
在连续的情形中，即
$$
\int_{-\infty}^x\int_{-\infty}^yp(u,v)dudv=\int_{-\infty}^xp_X(u)du\int_{-\infty}^yp_Y(v)dv=\int_{-\infty}^x\int_{-\infty}^yp_X(u)p_Y(v)dudv
$$
因为上式对于任意 $x,y$ 均成立，所以独立性条件可以用密度函数直接表示为
$$
\forall (x,y)\in \mathbb{R}^2,p(x,y)=p_X(x)p_Y(y)
$$

### n-Dimensional Continuous Random Variable

$\fbox{Definition 9.2}$ 设 $n$ 维随机变量 $(X_1,\cdots, X_n)$ 的分布函数为 $F(x_1,\cdots, x_n)$，若存在非负可积函数 $p(x_1,\cdots, x_n)$ 使得
$$
\forall (x_1,\cdots,x_n)\in \mathbb{R}^n,F(x_1,\cdots, x_n)=\int_{-\infty}^{x_1}\cdots \int_{-\infty}^{x_n} p(u_1,\cdots, u_n)du_1\cdots du_n
$$
则称 $(X_1,\cdots, X_n)$ 为 $n$ 维连续型随机变量，$p(x_1,\cdots, x_n)$ 为 (联合) 概率密度函数。                                                                                                                                                                                                                                                                                                                                                                                                                      

## Examples of 2-D Continuous Random Variable 

【例】 (二维均匀分布) 设 $D\subset \mathbb{R}^2$ 为有界区域，面积为 $S_D$。若 $(X,Y)$ 的联合密度函数为
$$
p(x,y)=\begin{cases}
\frac{1}{S_D}&, (X,Y)\in D\\\\
0&, (X,Y)\in D
\end{cases}
$$
则称 $(X,Y)$ 服从区域 $D$ 上的二维均匀分布。

(注：事实上该定义和一维情况相同，一维情况的测度是长度，二维情况的测度是面积。)

该分布的均匀性体现在：对于任意 $A\subseteq D$，若 $A$ 的面积为 $S_A$，则 $P((X,Y)\in A)=\frac{S_A}{S_D}$，与 $A$ 的形状、位置无关，只与 $A$ 的面积有关 (类比二维几何概型)。