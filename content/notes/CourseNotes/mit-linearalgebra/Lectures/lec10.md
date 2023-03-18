---
title: "MIT-18.06 Lecture 10: 4 Fundamental Subspaces"
linktitle: 'Lecture 10'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 10
---

4 fundamental subspaces:

* column space $C(A)$.
* null space $N(A)$.
* row space. It contains all the combinations of the row vectors of $A$. In other words, it contains all the combinations of the column vectors of $A^T$, so the notation is $C(A^T)$.
* left null space. It's the null space of $A^T$, denoted as $N(A^T)$.

If $A$ is a matrix with size $m\times n$, then $C(A),N(A^T)\subseteq \mathbb{R}^m$ while $C(A^T),N(A)\subseteq \mathbb{R}^n$. We've known that $\dim C(A)=r,\dim N(A)=n-r$. Here an amazing fact is that $\dim C(A)=\dim C(A^T)=r$. Thus $\dim N(A^T)=m-r$. The beautiful property is that $\dim C(A)+\dim N(A^T)=m$ and $\dim C(A^T)+\dim N(A)=n$, which corresponds to the fact that the number of pivot variables pluses the number of free variables equals $n$.

About basis, the pivot columns are a set of basis of $C(A)$ and the special solutions of $A\mathbf{x}=\mathbf{0}$ form a basis of $N(A)$. 

Of course we can transpose $A$ and do row operations on $A^T$ to get a basis of $C(A^T)$. But here's an easier way: we can get a basis of $C(A^T)$ through the reduced row echelon form. Let's take an example:
$$
A=\left[\begin{matrix}1 & 2 & 3 & 1\\\\1 & 1 & 2 & 1\\\\1 & 2 & 3 & 1\end{matrix}\right]
\rightarrow
U=\left[\begin{matrix}1 & 2 & 3 & 1\\\\0 & -1 & -1 & 0\\\\0 & 0 & 0 & 0\end{matrix}\right]
\rightarrow
R=\left[\begin{matrix}1 & 0 & 1 & 1\\\\ 0 & 1 & 1 & 0\\\\0 & 0 & 0 & 0\end{matrix}\right]
$$
Through $R$, we can see that the first two columns are pivot columns, so the first two columns of $A$ are basis of $C(A)$. Here we do row operations on $A$, so $C(A)\neq C(R)$ of course, but **considering the fact that doing row operations is just doing linear combinations of row vectors of $A$, $C(A^T)=C(R^T)$.** And we can easily tell that $\dim R^T=2=r$ and **the first $r$ row vectors of $R$ are basis of $C(A^T)$.** (They're independent and if we do the row operations inversely, we can get all the rows of $A$)

To find out a basis for $N(A^T)$. Firstly we want to figure out the matrix that represents the row operations of $A\rightarrow R$, i.e. we want to find a matrix $E$ such that $EA=R$.

According to Gauss-Jordan, we put a identity matrix besides $A$, i.e. $A'=[A_{m\times n}\quad I_{m\times m}]$. Suppose $EA'=$

$E[A\quad I]=[R\quad E']$, then we have $E=E'$, so by applying all the row operations on $I$, we can get $E$. In the above example, we can get
$$
E=\left[\begin{matrix}-1 & 2 & 0\\\\1 & -1 & 0\\\\-1 & 0 & 1\end{matrix}\right]
$$
We claim **that the last $m-r$ row vectors of $E$ form a basis of $N(A^T)$** because the last $m-r$ lines of $R$ are zero lines.

