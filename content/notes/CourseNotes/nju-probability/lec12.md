---
title: " Lecture 12: More Examples of 2-Dimensional Random Variable Function"
linktitle: Lecture 12
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
        weight: 12

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
--- 

【例】 (差的分布) 设 $(X,Y)$ 的密度为 $p(x,y)$，考虑 $Z=X-Y$ 的分布：
$$
\begin{align}
F_Z(z)&=P(X-Y\leq z)=\iint_{\{(x,y):x-y\leq z\}}p(x,y)dxdy\\\\
&=\int_{-\infty}^{+\infty}\left(\int_{x-z}^{+\infty}p(x,y)dy\right)dx\\\\
&\overset{y=x-v}{=}\int_{-\infty}^{+\infty}\int_z^{-\infty}p(x,x-v)d(-v)dx\\\\
&=\int_{-\infty}^z\left(\int_{-\infty}^{+\infty}p(x,x-v)dx\right)dv
\end{align}
$$
因此 $Z$ 的密度函数为
$$
p_Z(v)=\int_{-\infty}^{+\infty}p(x,x-v)dx
$$
如果先积 x，可以类似地得到另一种表达式：$p_Z(u)=\int_{-\infty}^{+\infty}p(y+u,y)dy$。

【例题】 (习题 3.27) 设 $(X,Y)$ 的密度为 $p(x,y)=\begin{cases}3x&,0<y<x<1\\\\0&,otherwise\end{cases}$，求 $Z=X-Y$ 的密度。

> 解：画图容易得出 $p(x,y)$ 的有效区域是一个三角形。
> $$
> F_Z(z)=P(X-Y\leq z)=\iint_{\{(x,y):x-y\leq z\}}p(x,y)dxdy
> $$
> 用直线 $x=y+z$ 滑过平面，关注直线的左侧。容易看出 $z<0$ 时 $F_Z(z)=0$，$z\geq 1$ 时 $F_Z(z)=1$。这两中情况下 $p_Z(z)=0$。$0\leq z<1$ 时，区域不太规则，要分成两个部分：
> $$
> \begin{align}
> F_Z(z)&=\int_0^z\int_0^x3xdydx+\int_z^1\int_{x-z}^x3xdydx\\\\
> &=\int_0^z3x^2dx+\int_z^13zxdx\\\\
> &=-\frac{1}{2}z^3+\frac{3}{2}z
> \end{align}
> $$
> 从而 $p_Z(z)=F_Z'(z)=-\frac{3}{2}z^2+\frac{3}{2}$。

【例】 (积和商的分布) 设 $(X,Y)$ 的密度为 $p(x,y)$，考虑两者积和商的分布：

* $Z=XY$
* $Z=\frac{X}{Y}$。

对于积的分布，
$$
F_Z(z)=P(XY\leq z)=\iint_{\{(x,y):xy\leq z\}}p(x,y)dxdy
$$
我们要小心 $x$ 的符号对积分上下限的影响，因此要分类讨论 $x<0$ 和 $x>0$ 的情况：
$$
\begin{align}
F_Z(z)&=\int_{-\infty}^0\left(\int_{z/x}^{+\infty} p(x,y)dy\right)dx+\int_0^{+\infty}\left(\int_{-\infty}^{z/x}p(x,y)dy\right)dx\\\\
&\overset{y=\frac{v}{x}}{=}\int_{-\infty}^0\left(\int_z^{-\infty}p(x,\frac{v}{x})d\frac{v}{x}\right)
dx+\int_0^{+\infty}\left(\int_{-\infty}^zp(x,\frac{v}{x})d\frac{v}{x}\right)dx\\\\
&=\int_{-\infty}^z\int_{-\infty}^0-p(x,\frac{v}{x})\frac{1}{x}dxdv+\int_{-\infty}^z\int_0^{+\infty}p(x,\frac{v}{x})\frac{1}{x}dxdv\\\\
&=\int_{-\infty}^z\left(\int_{-\infty}^{+\infty}p(x,\frac{v}{x})\frac{1}{|x|}dx\right)dv
\end{align}
$$
因此密度函数
$$
p_Z(z)=\int_{-\infty}^{+\infty}p(x,\frac{v}{x})\frac{1}{|x|}dx
$$
当然，如果先积 $y$ 再积 $x$，我们可以得到对称的形式：$p_Z(u)=\int_{-\infty}^{+\infty}p(\frac{u}{y},y)\frac{1}{|y|}dy$。

对于商的分布，
$$
F_Z(z)=P(X/Y\leq z)=\iint_{\{(x,y):\frac{x}{y}\leq z\}}p(x,y)dxdy
$$
类似地，考虑 $y<0$ 和 $y>0$，
$$
\begin{align}
F_Z(z)&=\int_{-\infty}^0\left(\int_{yz}^{+\infty}p(x,y)dx\right)dy+\int_0^{+\infty}\left(\int_{-\infty}^{yz}p(x,y)dx\right)dy\\\\
&\overset{x=yu}{=}\int_{-\infty}^0\left(\int_z^{-\infty}p(yu,y)d(yu)\right)dy+\int_0^{+\infty}\left(\int_{-\infty}^zp(yu,y)d(yu)\right)dy\\\\
&=\int_{-\infty}^z\left(\int_{-\infty}^0p(yu,y)(-y)dy\right)du+\int_{-\infty}^z\left(\int_0^{+\infty}p(yu,y)ydy\right)du\\\\
&=\int_{-\infty}^z\left(\int_{-\infty}^{+\infty}p(yu,y)|y|dy\right)du
\end{align}
$$
因此密度函数
$$
p_Z(u)=\int_{-\infty}^{+\infty}p(yu,y)|y|dy
$$
【例题】 (习题 3.28) $(X,Y)$ 的密度为 $p(x,y)=\begin{cases}\frac{1}{2}&,0\leq x\leq 2,0\leq y\leq 1\\\\0&,otherwise\end{cases}$，求 $Z=XY$ 的密度。

>解：用反比例函数曲线 $z=xy$ 滑过平面，考虑曲线下方的部分。显然当 $z<0$ 时 $F_Z(z)=0$，$z\geq 2$ 时 $F_Z(z)=1$。下面考虑 $0\leq z<2$ 的部分：
>$$
>F_Z(z)=\int_0^z\int_0^1\frac{1}{2}dydx+\int_z^2\int_0^{\frac{z}{x}}\frac{1}{2}dydx=\frac{z}{2}(\ln 2+1-\ln z)
>$$
>从而 $p_Z(z)=F_Z'(z)=\frac{1}{2}(\ln 2-\ln z)$。