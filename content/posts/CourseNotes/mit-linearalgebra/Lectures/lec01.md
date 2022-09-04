---
title: "MIT-18.06 Lecture 01: The Geometric Interpretation of Equations"
linktitle: 'Lecture 01'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 1
---

Consider 2 equations with 2 unknowns. Here's an example:
$$
\begin{cases}
2x-y=0\\\\
-x+2y=3
\end{cases}
$$
In linear algebra, we have $A=\left [\begin{matrix}2 & -1\\\\-1 & 2\end{matrix}\right ]$ the coefficient matrix, vector $\mathbf{x}=\left(\begin{matrix}x\\\\y\end{matrix}\right)$,  $\mathbf{b}=\left(\begin{matrix}0\\\\3\end{matrix}\right)$. Then the equations can be written in the form
$$
A\mathbf{x}=\mathbf{b}
$$
If we focus on the "column picture", the equations can be regarded as **linear combinations** of vectors
$$
x\begin{bmatrix}2 \\\\-1\end{bmatrix}+y\begin{bmatrix}-1\\\\2\end{bmatrix}=\begin{bmatrix}0\\\\3\end{bmatrix}
$$
The column picture of the solution to the equations is shown below:

<img src="/img/mit1806-lec01.png" alt="image-20210826223648690" style="zoom:33%;" />

Under the idea of linear combinations, the question

> Can I solve $A\mathbf{x}=\mathbf{b}$ for every $\mathbf{b}\in \mathbb{R}^n$ ?

is equivalent to the question

> Do the combinations of the columns fill the n-dimensional space?

The answer is not always "yes". For instance, if one vector is the combination of another 2 vectors, the combinations of the columns cannot fill the whole space. This case is called a **singular** case and the coefficient matrix $A$ is **not invertible**.

---

The product of a matrix and a vector
$$
\left [\begin{matrix}a & b\\\\c & d\end{matrix}\right]\left (\begin{matrix}x\\\\y\end{matrix}\right)=\left (\begin{matrix}ax+by\\\\cx+dy\end{matrix}\right)
$$
can be understood as
$$
\left [\begin{matrix}a & b\\\\c & d\end{matrix}\right]\left (\begin{matrix}x\\\\y\end{matrix}\right)=x\left(\begin{matrix}a\\\\c\end{matrix}\right)+y\left(\begin{matrix}b\\\\d\end{matrix}\right)
$$
which means that $A\mathbf{x}$ can be interpreted as the linear combinations of vectors.