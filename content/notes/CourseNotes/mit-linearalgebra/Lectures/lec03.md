---
title: "MIT-18.06 Lecture 03: Multiplication and Inverse Matrix"
linktitle: 'Lecture 03'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 3
---

Suppose $AB=C$ and $A,B,C$ are matrices, the formula for calculating a single entry of $C$ is
$$
C_{i,j}=(row_i\space of\space A)\cdot(col_j\space of\space B)=\sum_{k=1}^nA_{i,k}B_{k,j} 
$$
To apply matrix multiplication, we must ensure that the number of columns of $A$​ equals the number of row s of $B$​. If $A$​ is $m\times n$​ in size and $B$​ is $n\times p$​ in size, then C is $m\times p$​ in size.

According to what we’ve covered in the last two lectures, if we divide $B$ into several columns, then $C$ can be interpreted as several linear combinations of column vectors of $A$ sitting together. Similarly, if we divide $A$ into several rows, $C$​ can be interpreted as several linear combinations of row vectors of $B$ sitting together.

The fourth way of doing matrix multiplication is
$$
C=\sum_{k=1}^n(col_i\space of A)\times (row_i\space of\space B)
$$
notice that here we use the cross product. The matrix produced by $(col_i\space of\space A)\times (row_i\space of\space B)$ has some good properties: each column of the result matrix is multiples of $(col_i\space of\space A)$ and each row of the result matrix is multiples of $(row_i\space of\space B)$. In other words, the row space and column space of the result matrix are both a single line. 

The fifth way is block multiplication. For example, if $A$ and $B$ are square matrices, we divide $A$ and $B$ into four blocks, then
$$
C=\left[\begin{matrix}C_1 & C_2\\\\C_3 & C_4\end{matrix}\right]
=\left[\begin{matrix}A_1 & A_2\\\\A_3 & A_4\end{matrix}\right]
\left[\begin{matrix}B_1 & B_2\\\\B_3 & B_4\end{matrix}\right]
$$
$C_1=A_1B_1+A_2B_3$ ,here the multiplication are all matrix multiplication. Other blocks follow the same rule.

---

For convenience, let’s focus on the inverses of square matrices.

If a square matrix $A$ has an inverse, then $A^{-1}A=I$. Here for square matrices, the left inverse equals to the right inverse, so $AA^{-1}$ will also give $I$​.

Let’s consider the cases that a square matrix has no inverse, which we call it singular or non-invertible.
$$
A=\left[\begin{matrix}1 & 3\\\\2 & 6\end{matrix}\right]
$$
Why doesn’t it have an inverse? Suppose it has an inverse matrix, consider the linear combinations of the row vectors of $A$. The two vectors are collinear, so it’s impossible to make the linear combinations equals $(1,0)$ and $(0,1)$.

We can consider it in another way: **We can find a non-zero vector $\mathbf{x}$ such that $A\mathbf{x}=0$​**. For this example, $\mathbf{x}=\left(\begin{matrix}3\\\\-1\end{matrix}\right)$ is a suitable column vector. If there exists a non-zero vector $\mathbf{x}$ such that $A\mathbf{x}=0$, then suppose $A$​ has an inverse, then $A^{-1}(A\mathbf{x})=(A^{-1}A)\mathbf{x}=I\mathbf{x}=\mathbf{x}\neq 0$，however, $A^{-1}\cdot 0=0$, which leads to a contradiction.

The two ways describe the same thing: **If one of the row/column is useless, i.e. it can be constructed by other rows/columns, the matrix is non-invertible.**

---

For the invertible/non-singular matrices, how to calculate the inverse? Here we introduce the Gauss-Jordan method.

Consider an invertible $2\times 2$ square matrix
$$
\left[\begin{matrix}1 & 3\\\\2 & 7\end{matrix}\right]
$$
let’s put the identity matrix besides it to make it an augmented matrix.
$$
\left[\begin{matrix}1 & 3 & 1 & 0\\\\2 & 7 & 0 & 1\end{matrix}\right]
$$
Let’s do eliminations on the left matrix. Different from the classic Gauss elimination whose target is to get the upper triangle matrix,here we need to get the identity matrix. To implement that, after we choose a pivot, we should eliminate all the elements that are in the same column and not in the current row, including the elements above.
$$
\left[\begin{matrix}1 & 3 & 1 & 0\\\\2 & 7 & 0 & 1\end{matrix}\right]\overset{(2,1)}{\rightarrow}\left[\begin{matrix}1 & 3 & 1 & 0\\\\0 & 1 & -2 & 1\end{matrix}\right]
\overset{(1,2)}{\rightarrow}\left[\begin{matrix}1 & 0 & 7 & -3\\\\0 & 1 & -2 & 1\end{matrix}\right]
$$
The matrix on the right $\left[\begin{matrix}7 & -3\\\\-2 & 1\end{matrix}\right]$ is the inverse.

The principle is easy: according to the last lectures, elimination steps can be interpreted as several elimination matrices. Consider all the elimination steps give us $E$, then $EA=I$, which means that $E=A^{-1}$. The right matrix is $EI=E=A^{-1}$, so we get the right solution.