---
title: "MIT-18.06 Lecture 04 - A’s LU Decomposition"
linktitle: 'Lecture 04'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 4
---

What’s the inverse of $AB$, supposing $A$ and $B$​ are both invertible matrices? We can use $B^{-1}A^{-1}$ to check:
$$
\begin{align}
(B^{-1}A^{-1})AB=B^{-1}(A^{-1}A)B=B^{-1}B=I\\\\
AB(B^{-1}A^{-1})=A(BB^{-1})A^{-1}=AA^{-1}=I
\end{align}
$$
So $(AB)^{-1}=B^{-1}A^{-1}$.

What’s the inverse of $A^T$? Consider $AA^{-1}=I$. Transpose both sides and we get
$$
(AA^{-1})^T=(A^{-1})^TA^T=(I)^T=I
$$
so $(A^T)^{-1}=(A^{-1})^T$. In other words, you can change the order of inversing and transposing.

---

Consider a $2\times 2$ matrix
$$
A=\left[
\begin{matrix}
	2 & 1\\\\
	8 & 7
\end{matrix}
\right]
$$
To eliminate it to an upper triangle matrix $U$， we use the elementary matrix
$$
E_{2,1}=\left[\begin{matrix}1 & 0\\\\-4 & 1\end{matrix}\right],E_{2,1}A=U=\left[\begin{matrix}2 & 1\\\\0 & 3\end{matrix}\right]
$$
To write it in the form of $A=LU$，we can easily find that $L=E^{-1}$ and for elementary matrices, the inverse is easy to calculate - just add a negative sign '-'. So $L=\left[\begin{matrix}1 & 0\\\\4 & 1\end{matrix}\right]$. Here $L$ represents lower triangle matrix.
$$
A=LU=\left[\begin{matrix}1 & 0\\\\4 & 1\end{matrix}\right]\left[\begin{matrix}2 & 1\\\\0 & 3\end{matrix}\right]
$$


In some cases, we write
$$
A=LDU=\left[\begin{matrix}1 & 0\\\\4 & 1\end{matrix}\right]\left[\begin{matrix}2 & 0\\\\0 & 3\end{matrix}\right]\left[\begin{matrix}1 & \frac{1}{2}\\\\0 & 1\end{matrix}\right]
$$
We extract the diagonal matrix $D$ from $U$ to makes it cleaner.

Here we cannot see differences between $EA=U$ and  $A=LU$. Let's consider a $3\times 3$ example. Say there's no row exchanges, $E_{3,2}E_{3,1}E_{2,1}A=U$，$E_{3,1}$ is an identity matrix and 
$$
E_{3,1}=\left[\begin{matrix}1 & 0 & 0\\\\-2 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
,
E_{3,2}=\left[\begin{matrix}1 & 0 & 0\\\\0 & 1 & 0\\\\0 & -5 & 1\end{matrix}\right]
$$
so
$$
E=E_{3,2}E_{3,1}E_{2,1}=
\left[\begin{matrix}1 & 0 & 0\\\\-2 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
\left[\begin{matrix}1 & 0 & 0\\\\0 & 1 & 0\\\\0 & -5 & 1\end{matrix}\right]
= \left[\begin{matrix}1 & 0 & 0\\\\-2 & 1 & 0\\\\10 & -5 & 1\end{matrix}\right]
$$
It's not obvious to tell why the number $10$ appears. Actually it's because that we firstly subtract $2\times row_1$ from $row_2$，then we subtract $5\times (new)row_2$ from $row_3$, and in total we add $10\times row_1$ to $row_3$.

If we use the $A=LU$ form, then
$$
L=E_{2,1}^{-1}E_{3,1}^{-1}E_{3,2}^{-1}=\left[\begin{matrix}1 & 0 & 0\\\\2 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]\left[\begin{matrix}1 & 0 & 0\\\\0 & 1 & 0\\\\0 & 5 & 1\end{matrix}\right]=\left[\begin{matrix}1 & 0 & 0\\\\2 & 1 & 0\\\\0 & 5 & 1\end{matrix}\right]
$$
We can see that due to the elaborate order, operations will not interfere with each other. **If there's no row exchange, the multipliers during eliminations will go directly into $L$.** 

That's the advantage of $A=LU$. With $L$ and $U$ we can completely forget about $A$ since $LU$ contains all the information about $A$. What's more, $L$ , the inverse of $E$, has the beautiful property that it contains the complete information about elimination steps.

> **How many operations on $A$ during elimination?**
>
> For a $n\times n$ matrix, it takes us approximately $n^2$ operations to tackle with the first row and the first column, then the problem becomes a $(n-1)\times (n-1)$ one. So the cost is
> $$
> \sum_{k=1}^nk^2\sim \int_1^nx^2\mathrm{d}x\sim \frac{1}{3}n^3
> $$
> And for a column vector, say $\mathbf{b}$, the cost is approximately $n^2$.

