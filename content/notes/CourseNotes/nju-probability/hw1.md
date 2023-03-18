---
title: "Homework 1"
linktitle: Homework 1
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
        parent: Homework
        weight: 1

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
---

## Problem 1.1

解：(1) $A_1\cap \overline{A_2}\cap \overline{A_3}$。

(2) $A_1\cup A_2\cup A_3$。

(3) $(A_1\cap \overline{A_2}\cap \overline{A_3})\cup (\overline{A_1}\cap A_2\cap \overline{A_3})\cup (\overline{A_1}\cap \overline{A_2}\cap A_3)\cup (\overline{A_1}\cap \overline{A_2}\cap \overline{A_3})$。 

(4) $\overline{A_1\cap A_2\cap A_3}$。

(5) $(A_1\cap A_2)\cup(A_1\cap A_3)\cup (A_2\cap A_3)$。

## Problem 1.3

解：
$$
P=\frac{\binom{10}{4}\binom{4}{3}\binom{3}{2}}{\binom{17}{9}}=\frac{252}{2431}。
$$

## Problem 1.7

解：
$$
P=\frac{9^n-5^n-8^n+4^n}{9^n}=1-\frac{5^n+ 8^n-4^n}{9^n}
$$

## Problem 1.10

解：记第一天下雨为事件 $A$，第二天下雨为事件 $B$。

(1) $P(至少有一天下雨)=P(A\cup B)=P(A)+P(B)-P(A\cap B)=0.6+0.3-0.1=0.8$。

(2) $P(两天都不下雨)=P(\overline{A}\cap \overline{B})=P(\Omega)-P(A\cup B)=1-0.8=0.2$。

(3) $P(至少有一天不下雨)=P(\overline{A}\cup \overline{B})=P(\Omega)-P(A\cap B)=1-0.1=0.9$。

(4) $P(第一天下雨且第二天不下雨)=P(A\cap \overline{B})=P(A)-P(AB)=0.6-0.1=0.5$。

(5) $P(恰好一天下雨)=P(A)+P(B)-2P(AB)=0.6+0.3-0.1-0.1=0.7$。

## Problem 1.13

解：容易发现三条折线能构成三角形 $\Leftrightarrow$ 三条折线的长度均小于 $a$。

设第一处折点距离线段左端点的距离为 $x$，第二处折点距离线段左端点的距离为 $y$，则上述约束可以被翻译为：
$$
\begin{cases}
x<y\\\\
x<a\\\\
y-x<a\\\\
y>a
\end{cases}
$$
画图：

<img src="Homework 01.assets/image-20220312152059033.png" alt="image-20220312152059033" style="zoom: 33%;" />

容易看出，$P(能构成三角形)=0.25$。

## Problem 1.15

解：$P(AB)=P(A)P(B|A)=\frac{1}{4}\cdot \frac{1}{3}=\frac{1}{12}$，所以 $P(B)=\frac{P(AB)}{P(A|B)}=\frac{1}{6}$。

$P(\bar{A}\bar{B})=P(\Omega-A\cup B)=1-(P(A)+P(B)-P(AB))=1-\frac{1}{4}-\frac{1}{6}+\frac{1}{12}=\frac{2}{3}$。

## Problem 1.18

解：记取了甲车间产品为事件 $A$，乙为事件 $B$，丙为事件 $C$，取到次品为事件 $D$。

(1) $P(D)=P(AD)+P(BD)+P(CD)=0.25\cdot 0.05+0.35\cdot 0.04+0.4\cdot 0.02=0.0345$。

(2) $P(A|D)=\frac{P(AD)}{P(D)}=\frac{P(A)P(D|A)}{P(D)}=\frac{0.25\cdot 0.05}{0.0345}=\frac{25}{69}$。

## Problem 1.21

解：一共有 3 个面是红色的，这三个红色的面中只有 1 个面背后是黄色的，所以 $P=\frac{1}{3}$。

## Problem 1.24

解：因为 $A,B$ 相互独立，所以 $P(AB)=P(A)P(B)$。我们列出方程：
$$
\begin{cases}
P(A)-P(A)P(B)&=\frac{5}{9}\\\\
P(A)+P(B)-P(A)P(B)&=\frac{8}{9}\\\\
0\leq P(A),P(B)\leq 1
\end{cases}
$$
解得
$$
\begin{cases}
P(A)=\frac{5}{6}\\\\
P(B)=\frac{1}{3}
\end{cases}
$$
综上，$P(A)=\frac{5}{6}$。

## Problem 1.29

解：$P(A)=\frac{3}{6}=\frac{1}{2},P(B)=\frac{3}{6}=\frac{1}{2},P(C)=\frac{18}{36}=\frac{1}{2}$。

显然 $P(ABC)=0\neq P(A)P(B)P(C)$。

但
$$
\begin{align}
P(AB)&=\frac{3\times 3}{6\times 6}=\frac{1}{4}=P(A)P(B)\\\\
P(AC)&=P(A\cap \overline{B})=\frac{9}{36}=\frac{1}{4}=P(A)P(C)\\\\
P(BC)&=P(B\cap \overline{A})=\frac{9}{36}=\frac{1}{4}=P(B)P(C)
\end{align}
$$
所以这三个事件两两独立，但不相互独立。

## Problem 1.32

解：记两人命中数一样为事件 $A$，两人都中 $i$ 球为事件 $B_i$，则
$$
\begin{align}
P(A)&=P(A|B_0)+P(A|B_1)+P(A|B_2)\\\\
&=\left(\frac{1}{3}\right)^2\left(\frac{2}{3}\right)^2+\frac{4}{9}\cdot \frac{4}{9}+\left(\frac{2}{3}\right)^2\left(\frac{1}{3}\right)^2\\\\
&=\frac{8}{27}
\end{align}
$$

## Problem 1.34

解：记一个人蒙及格为事件 $A$。

(1)
$$
P(A)=\sum_{k=3}^5\binom{5}{k}\left(\frac{1}{4}\right)^k\left(\frac{3}{4}\right)^{5-k}=\frac{53}{512}
$$
(2) 
$$
\begin{align}
P(至少两人蒙及格)&=1-P(至多一人蒙及格)\\\\
&=1-\sum_{k=0}^1\binom{5}{k}P(A)^k(1-P(A))^{5-k}\\\\
&=0.0866
\end{align}
$$

