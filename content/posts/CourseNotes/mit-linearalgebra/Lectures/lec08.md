---
title: "MIT-18.06 Lecture 08: Complete Solutions of Equations"
linktitle: 'Lecture 08'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 8
---

Take an example:
$$
A\mathbf{x}=\left[\begin{matrix}1 & 2 & 2 & 2\\\\2 & 4 & 6 & 8\\\\3 & 6 & 8 & 10\end{matrix}\right]\left(\begin{matrix}x_1\\\\x_2\\\\x_3\end{matrix}\right)\left(\begin{matrix}b_1\\\\b_2\\\\b_3\end{matrix}\right)=\mathbf{b}
$$

$$
\left[\begin{matrix}1 & 2 & 2 & 2 & b_1\\\\2 & 4 & 6 & 8 & b_2\\\\3 & 6 & 8 & 10 & b_3\end{matrix}\right]
\overset{elimination}{\longrightarrow}
\left[\begin{matrix}1 & 2 & 2 & 2 & b_1\\\\0 & 0 & 2 & 4 & b_2-2b_1\\\\0 & 0 & 0 & 0 & b_3-b_2-b_1\end{matrix}\right]
$$

So the equations have solutions exactly when $b_3=b_1+b_2$.

Let's talk about the solvability. $A\mathbf{x}=\mathbf{b}$ is solvable exactly when $b\in C(A)$. Another equivalent description is that: **If a combination of rows of $A$ gives zero row, then the same combination of components of $\mathbf{b}$ must give 0.**

To find the complete solutions to $A\mathbf{x}=\mathbf{b}$:

0. Ensure that the zero rows = 0.
1. Find a particular solution $\mathbf{x_p}$: set all free variables to 0 and solve $A\mathbf{x}=\mathbf{b}$ for pivot variables.
2. Find $N(A)$, then the complete solution set is $X_c=\{\mathbf{x}=\mathbf{x_p}+\mathbf{x_n}:\mathbf{x_n}\in N(A)\}$

---

Let's focus on a bigger picture: an $m\times n$ matrix with rank $r$ (it's obvious that $r\leq m,r\leq n$):

* Case 1: Full column rank $(r=n<m)$

    No free variable, so $N(A)=\{\mathbf{0}\}$. The reduced row echelon form $R=\left[\begin{matrix}I\\\\0\end{matrix}\right]$.

    If the particular solution $\mathbf{x_p}$ exists, then it's the unique solution. The equations have 0 or 1 solutions.

* Case 2: Full row rank $(r=m<n)$

    We can solve $A\mathbf{x}=\mathbf{b}$ for arbitrary $\mathbf{b}$. The reduced row echelon form $R=\left[\begin{matrix}I & F\end{matrix}\right]$. (Notice that the columns of $I$ and $F$ may mix together)

    Since $F$ exists, $N(A)$ contains more than $\mathbf{0}$, the equations have $\infty$ solutions.

* Case 3: $r=n=m$

    In this case $R=I$, which indicates that $A$ is invertible. And the equations have a unique solution.

* Case 4: $r<n,r<m$

    The row reduced echelon form $R=\left[\begin{matrix}I & F\\\\0 & 0\end{matrix}\right]$. The equations have either no solution (zero rows don't match $\mathbf{b}$) or $\infty$ solutions.

One sentence to summarize:  **the rank $r$ tells you all the information about the number of solutions**.

