---
title: "MIT-18.06 Lecture 07: Computing Null Space, Pivot Variables and Special Solutions"
linktitle: 'Lecture 07'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 7
---

We solve the equation $A\mathbf{x}=\mathbf{0}$ through elimination. Obviously, elimination won't change the null space of $A$. For example, let
$$
A=\left[\begin{matrix}1 & 2 & 2 & 2\\\\2 & 4 & 6 & 8\\\\3 & 6 & 8 & 10\end{matrix}\right]
$$
through elimination we get the echelon matrix $U$:
$$
A=\left[\begin{matrix}1 & 2 & 2 & 2\\\\2 & 4 & 6 & 8\\\\3 & 6 & 8 & 10\end{matrix}\right]\overset{(2,1),(3,1)}{\longrightarrow}\left[\begin{matrix}1 & 2 & 2 & 2\\\\0 & 0 & 2 & 4\\\\0 & 0 & 2 & 4\end{matrix}\right]\overset{(3,3)}{\longrightarrow}\left[\begin{matrix}1 & 2 & 2 & 2\\\\0 & 0 & 2 & 4\\\\0 & 0 & 0 & 0\end{matrix}\right]=U
$$
The number of pivots is call the **rank** of $A$. There are two pivots, so $rank(A)=2$.

By convention, we choose 1st and 3rd column as pivot columns, and the rest are free columns. Accordingly, $x_1$ and $x_3$ are pivot variables, $x_2$ and $x_4$ are free variables. We can assign values to free variables freely, and for every set of values, there's a unique solution to the pivot variables.

In particular, if we assign $x_2=1,x_4=0$ , and $x_2=0,x_4=1$, we can get two solutions:
$$
\mathbf{x_1}=
\left(\begin{matrix}-2\\\\1\\\\0\\\\0\end{matrix}\right)
\qquad
\mathbf{x_2}=
\left(\begin{matrix}2\\\\0\\\\-2\\\\1\end{matrix}\right)
$$
These solutions are called **special solutions**. The linear combinations of the special solutions forms the null space. That is
$$
N(A)=c\left(\begin{matrix}-2\\\\1\\\\0\\\\0\end{matrix}\right)+d\left(\begin{matrix}2\\\\0\\\\-2\\\\1\end{matrix}\right),c,d\in \mathbb{R}
$$
And for a matrix $A$ with size $m\times n$, the number of special solutions is $(n-rank(A))$. 

We can work harder on $U$ to get the **reduced row echelon form**, denoted as $R$ or $rref(A)$. It eliminates values both above and below the pivots, and transform pivots to 1:
$$
U=\left[\begin{matrix}1 & 2 & 2 & 2\\\\0 & 0 & 2 & 4\\\\0 & 0 & 0 & 0\end{matrix}\right]\overset{(1,3)}{\longrightarrow}\left[\begin{matrix}1 & 2 & 0 & -2\\\\0 & 0 & 1 & 2\\\\0 & 0 & 0 & 0\end{matrix}\right]=R
$$
To make it clear, we do column exchanges so that we put pivots columns together on the left and free columns together on the right:
$$
\left[\begin{matrix}1 & 2 & 0 & -2\\\\0 & 0 & 1 & 2\\\\0 & 0 & 0 & 0\end{matrix}\right]\overset{2\leftrightarrow 3}{\longrightarrow}\left[\begin{matrix}1 & 0 & 2 & -2\\\\0 & 1 & 0 & 2\\\\0 & 0 & 0 & 0\end{matrix}\right]
$$
The advantage of $R$ is that we can directly see the special solutions from the matrix. If we ignore the rows with all 0s, we find that $R=\left[\begin{matrix}I \mid F\end{matrix}\right]$.  $I$ is an identity matrix with size $r\times r$ ( $r$ means the rank) and $F$ is $r\times (n-r)$ in size. We can construct all the $(n-r)$ special solutions as a matrix:
$$
N=\left[\begin{matrix}-F\\\\I\end{matrix}\right]=\left[\begin{matrix}-2 & 2\\\\0 & -2\\\\1 & 0\\\\0 & 1\end{matrix}\right]
$$
$N$ is $n\times (n-r)$ in size. (Here the $I$ may have the different size from the $I$ in $R$) If we use block multiplication of matrices, we can find out that $RN$ gives a matrix with all 0s.

