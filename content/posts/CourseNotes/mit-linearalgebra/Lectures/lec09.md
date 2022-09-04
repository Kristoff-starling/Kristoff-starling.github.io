---
title: "MIT-18.06 Lecture 09: Linear Dependence, Basis and Dimension"
linktitle: 'Lecture 09'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 9
---

suppose $A$ is $m\times n$ in size $(m<n)$, then the equation $A\mathbf{x}=\mathbf{0}$ has nonzero solutions because there will be at least one free variable.

Vectors $\mathbf{x_1},\mathbf{x_2},...,\mathbf{x_n}$ are **independent** if no combination gives zero vector. (except the zero combination)

i.e. for any $c_1,c_2,...,c_n$, $\sum_{i=1}^n c_i\mathbf{x_i}\neq \mathbf{0}$ ($c_i$ are not all 0s). Otherwise they're **dependent**.

If $\mathbf{v_1},\mathbf{v_2},...,\mathbf{v_n}$ are column vectors of matrix $A$

* They're independent if $N(A)=\{\mathbf{0}\}$. In this case $rank(A)=n$. (No free variable, the essence of free columns is that it belongs to the linear combinations of previous pivot columns)
* They're dependent if for some nonzero vector $\mathbf{c}$, $A\mathbf{c}=\mathbf{0}$. In this case $rank(A)<n$.

$n$ m-dimensional vectors $\mathbf{x_1},...,\mathbf{x_n}$ $(m<n)$ is absolutely dependent. That's because if we put the vectors together to form a matrix $A=\left[\mathbf{x_1}\space \mathbf{x_2} \cdots \mathbf{x_m}\right]$, $A\mathbf{x}=\mathbf{0}$ has nonzero solutions.

---

Vectors $v_1,v_2,..,v_l$ **span** a space means that the space consists of all the combinations of those vectors.

**Basis** for a vector space is a sequence of vectors $\mathbf{v_1},\mathbf{v_2},...,\mathbf{v_d}$ satisfying

* They're independent.
* They span the whole space.

(In other words, the number of vectors is exactly "right".)

$n$ vectors in $\mathbb{R}^n$ give basis if the $n\times n$ matrix with those columns is invertible.

> if they're independent, then every column is pivot column, so the reduced echelon form $R=I$. And eliminations can be represented as matrices. So the original matrix is invertible.

Given a space, every basis for the space has the same number of vectors, and we call this number the **dimension** of the space.

Take an example, 
$$
A=\left[\begin{matrix}1 & 2 & 3 & 1\\\\1 & 1 & 2 & 1\\\\1 & 2 & 3 & 1\end{matrix}\right]
$$
The column vectors are dependent because $\left(\begin{matrix}-1\\\\-1\\\\1\\\\0\end{matrix}\right)\in N(A)$. $rank(A)=2$ and $dimC(A)=2$. We find a important property: **the rank of a matrix $A$ equals the dimension of $C(A)$, $dimC(A)=r$**.

Column vectors 1 and 2 gives a basis of $C(A)$ and 1 and 3, 2 and 3 are basis too.

The two special solutions in $N(A)$ are
$$
\mathbf{x_1}=\left(\begin{matrix}-1\\\\-1\\\\1\\\\0\end{matrix}\right)\qquad\mathbf{x_2}=\left(\begin{matrix}-1\\\\0\\\\0\\\\1\end{matrix}\right)
$$
In fact, $span\{\mathbf{x_1},\mathbf{x_2}\}=N(A)$. We see that the dimension of $N(A)$ equals the number of free variables. That is $dimN(A)=n-r$.

