---
title: "MIT-18.06 Lecture 06: Column Space, Null space"
linktitle: 'Lecture 06'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 6
---

Suppose there are two vector spaces $S$ and $T$, their intersection $S\cap T$ is also a subspace.

> Proof: suppose $v,w\in S\cap T$ are vectors, then consider a linear combination $av+bw$, since $v,w\in S$, $av+bw\in S$ , also we have $av+bw\in T$, so $av+bw\in S\cap T$, which proves the theorem.

---

Consider
$$
A=\left[\begin{matrix}1 & 1 & 2\\\\2 & 1 & 3\\\\3 & 1 & 4\\\\4 & 1 & 5\end{matrix}\right]
$$
all the linear combinations of the three column vectors of $A$ forms a column space $C(A)$. It's a subspace of $\mathbb{R}^4$ since all the vectors have 4 dimensions.

We can find that the subspace cannot fill the whole $\mathbb{R}^4$. Let's consider the linear equations
$$
A\mathbf{x}=
\left[\begin{matrix}1 & 1 & 2\\\\2 & 1 & 3\\\\3 & 1 & 4\\\\4 & 1 & 5\end{matrix}\right]
\left(\begin{matrix}x_1\\\\x_2\\\\x_3\end{matrix}\right)
=\left(\begin{matrix}b_1\\\\b_2\\\\b_3\\\\b_4\end{matrix}\right)=\mathbf{b}
$$
There are four equations with three unknowns. The equations don't necessarily have solutions. We care about what b's allow the equations to be solved.

If we use linear combinations to understand matrix multiplications, we can find that $A\mathbf{x}$ is the linear combinations of the column vectors of $A$, so **$\mathbf{b}$ allows the equations to be solved exactly when $\mathbf{b}\in C(A)$**.

What's more, we can find that the third column vector makes no contribution: $col_1+col_2=col_3$. If we delete the third column vector, the column space won't be changed. So actually $C(A)$ is a 2-dimensional subspace of $\mathbb{R}^4$. We call the first two columns **pivot columns**.

(Actually we can choose any column and discard it, but by convention we choose the first several columns that are independent.)

---

The null space of a matrix $A$, denoted as $N(A)$, contains all the solutions to $A\mathbf{x}=\mathbf{0}$. For the example $A$, all the vectors $\left(\begin{matrix}c\\\\c\\\\-c\end{matrix}\right)$ are in the null space, $N(A)$ is a line in $\mathbb{R}^3$.

The null space is really a subspace because for any $A\mathbf{v}=\mathbf{0}$ and  $A\mathbf{w}=\mathbf{0}$, $A\mathbf{(v+w)}=A\mathbf{v}+A\mathbf{w}=\mathbf{0}$.

From column spaces and null spaces we can see that there are two ways to construct a subspace:

* use the linear combinations of several vectors to form a subspace.
* add constraints to linear equations to form a subspace.

