---
title: "MIT-18.06 Lecture 05: Permutation, Transpose and Vector Spaces"
linktitle: 'Lecture 05'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 5
---

Now let's allow row exchanges to come in during elimination, so we need permutation matrices. 

Permutation matrices can be regarded as identity matrices with some rows exchanged. For $n\times n$ matrix, there are $n!$ kinds of permutation matrices (the identity matrix is a special permutation matrix).

An important property of permutation matrices is that
$$
P^{-1}=P^T
$$
The property is easy to understand: only the $i_{th}$ row and the $i_{th}$ column have $1$ in the same position, so $P^TP=I$.

If we take row exchanges into consideration, the formula will be
$$
PA=LU
$$
where $P$ is one of the permutation matrices to make the pivots at the right positions.

---

Transpose is defined as follows:
$$
(A^T)_{ij}=A_{ji}
$$
**Symmetric matrices** are those satisfying $A^T=A$. It's easy to construct a symmetric matrix because  **for any matrix $R$, $R^TR$ is a symmetric matrix.**   ($(R^TR)^T=R^T(R^T)^{T}=R^TR$)

---

Vector spaces are bunches of vectors. There are two basic operations: vector addition and multiplying a scalar to a vector. There are some rules for these two operations (refer to the textbook). The most important one is:

**The result of addition and multiplication (i.e. linear combinations of vectors) should stay in the vector space.**

We can easily check that $\mathbb{R}^n$ is a vector space, it contains all the column vectors with $n$  real components, while the first quadrant of $\mathbb{R}^2$ is not a vector space because it's not closed under multiplication by all real numbers.

Let consider subspaces of $\mathbb{R}^2$. There are three types of subspaces:

* $\mathbb{R}^2$ itself
* Any line that crosses through $\left(\begin{matrix}0\\\\0\end{matrix}\right)$ (we call it $L$)
* Zero vector only (we call it $Z$)

The conclusion can be easily extended to $\mathbb{R}^n$.

---

Let's see how we can use matrices to produce vector spaces. Take the $3\times 2$ matrix
$$
\left[\begin{matrix}1 & 3\\\\2 & 3\\\\4 & 1\end{matrix}\right]
$$
as an example, the columns of the matrix (i.e. two column vectors $\left(\begin{matrix}1\\\\2\\\\4\end{matrix}\right)$ and $\left(\begin{matrix}3\\\\3\\\\1\end{matrix}\right)$) span a subspace in $\mathbb{R}^3$. In geometry, the subspace is a plane going through the origin in $\mathbb{R}^3$. 

In general, all the linear combinations of the column vectors of a matrix $A$ form a subspace called **column space, denoted as $C(A)$**. 