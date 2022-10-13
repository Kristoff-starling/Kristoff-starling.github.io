---
title: "问题求解 III - 06 题解"
linktitle: 'III-06 Sol'
type: docs
draft: false

math: true

menu:
    oj-solutions:
        parent: Contents
        weight: 306
---

## A. 最大内切球

### 题面描述

* 给定 $n$ 个半平面，求半平面围出的三维凸包的最大内切球的半径。
* $\sum n\leq 2500$。

### 题解

设球的半径为 $r$，则我们的约束是球心到每个平面的距离不小于 $r$。此处根据空间解析几何的知识，如果一个半平面形如 $ax+by+cz+d\leq 0$，那么将其单位化之后（即令 $cos\alpha=\frac{a}{\sqrt{a^2+b^2+c^2}},cos\beta=\frac{b}{\sqrt{a^2+b^2+c^2}},$$cos\gamma=\frac{c}{\sqrt{a^2+b^2+c^2}}$，将方程改写为 $cos\alpha\cdot x+cos\beta\cdot y+cos\gamma\cdot z+\rho=0$的形式），直接将点的坐标代入即可得到点到平面的距离，因此每个约束都是一个线性不等式，共有 4 个变量，我们要最大化的式子就是 $r$，用单纯形求解即可。

## B. 文理分科

### 题面描述

* 给定排成矩阵的 $n\times m$ 个学生，每个学生学文有 $a_{ij}$ 的收益，学理有 $b_{ij}$ 的收益。如果一个学生和他相邻的所有学生都学文/理，可以额外获得 $sa_{ij}/sb_{ij}$ 的收益。求最大收益。
* $n,m\leq 100$。

### 题解

考虑最小割，我们先假设所有人拿满了学文学理的收益和区域相同的收益，考虑最少需要扣除多少收益。

* 建立超级源点 $S$ 和超级汇点 $T$。

* 对于每个学生，从 $S$ 向他连容量为 $a_{ij}$ 的边，表示如果割去，他去学理，则会损失该部分收益。相似地，从他向 $T$ 连容量为 $b_{ij}$ 的边。
* 对于每个学生和其相邻人同时学文的情况，我们需要扣除该部分收益当且仅当这些人中至少有一个学理。因此我们从 $S$ 向该学生的一个“副本点”连容量为 $sa_{ij}$ 的边，再从该副本点向该学生及其相邻人连容量为 $\infty$ 的边。这样这些人中一旦有一个人学理，就不得不割去容量为 $sa_{ij}$ 的边。$sb$ 的处理方法类似。

## C. Did You Just Say ...... Acesrc?

### 题面描述

* 给定一个 $2\times n$ 的“梯子”图，求该图有多少个连通的子图。
* $n\leq 10^{18}$。

### 题解

考虑动态规划。如果从左向右考虑，我们发现一个性质：到第 $i$ 列时，之前的点要么是连通的，要么分属两个连通块，且第 $i$ 列的两个点不连通。令 $f(i)$ 表示当前考虑到第 $i$ 列，只有一个连通块的方案数，$g(i)$ 表示当前考虑到第 $i$ 列，有两个连通块的方案数。稍微考虑以下第 $i$ 个“梯子格子”中的三条边，转移是容易的：
$$
\begin{align}
f(i)=4f(i-1)+g(i-1)\\\\
g(i)=2f(i-1)+g(i-1)
\end{align}
$$注意到该动态规划是线性递推，因此可以将 $f$ 和 $g$ 写成一个列向量，用矩阵表示转移，从而用矩阵快速幂优化转移。总时间复杂度 $O(logn)$。
