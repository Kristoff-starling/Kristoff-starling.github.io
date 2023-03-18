---
title: "MIT-18.06 Lecture 02: Elimination Matrices"
linktitle: 'Lecture 02'
type: docs
draft: false

math: true

menu:
    mit1806-lec:
        parent: Contents
        weight: 2
---

Consider the 3 equations with 3 unknowns:
$$
\begin{cases}
x+2y+z = 2\\\\
3x+8y+z = 12\\\\
4y+z = 2
\end{cases}
$$
The coefficient matrix
$$
A=\left[
\begin{matrix}
1 & 2 & 1\\\\
3 & 8 & 1\\\\
0 & 4 & 1
\end{matrix}
\right]
$$
The process of elimination is
$$
\left[
\begin{matrix}
1 & 2 & 1\\\\
3 & 8 & 1\\\\
0 & 4 & 1
\end{matrix}
\right]
\overset{(2,1)}{\rightarrow}
\left[
\begin{matrix}
1 & 2 & 1\\\\
0 & 2 & -2\\\\
0 & 4 & 1
\end{matrix}
\right]
\overset{(3,2)}{\rightarrow}
\left[
\begin{matrix}
1 & 2 & 1\\\\
0 & 2 & -2\\\\
0 & 0 & 5
\end{matrix}
\right]
$$
The basic procedure is repeatedly choosing pivots (Pivots cannot be zero) and using the pivot to eliminate other lines. The final target is the upper triangle matrix $U$. 

**When a non-zero pivot cannot be chosen from the currently available equations, the elimination failed.**

To find the solution to the equations, we need to apply the elimination steps on the **augmented matrix**. Augmented matrix means the coefficient matrix with an extra column.
$$
\left[
\begin{matrix}
1 & 2 & 1 & 2\\\\
3 & 8 & 1 & 12\\\\
0 & 4 & 1 & 2
\end{matrix}
\right]
\overset{(2,1)}{\rightarrow}
\left[
\begin{matrix}
1 & 2 & 1 & 2\\\\
0 & 2 & -2 & 6\\\\
0 & 4 & 1 & 2
\end{matrix}
\right]
\overset{(3,2)}{\rightarrow}
\left[
\begin{matrix}
1 & 2 & 1 & 2\\\\
0 & 2 & -2 & 6\\\\
0 & 0 & 5 & -10
\end{matrix}
\right]
$$

Typically, we call the target of $\mathbf{b}$​​​ the vector $\mathbf{c}$​​​. The target of elimination is to transform $A\mathbf{x}=\mathbf{b}$​ into $U\mathbf{x}=\mathbf{c}$. After back substitution, the solution is $x=2, y=1,z=-2$​​​.

---

In the last lecture we mentioned that a matrix multiplying a column vector can be interpreted as linear combinations of the column vectors of the matrix. In the same way,

$$
(x,y,z)\begin{bmatrix}a_1 & a_2 & a_3\\\\b_1 & b_2 & b_3\\\\c_1 & c_2 & c_3\end{bmatrix}=x\cdot (a_1,a_2,a_3)+y\cdot (b_1,b_2,b_3)+z\cdot (c_1,c_2,c_3)
$$

a row vector multiplying a matrix can be interpreted as linear combinations of the row vectors of the matrix.

So we can use “matrix language” to explain the above elimination steps.

Step 1: Subtract $3\times row_1$ from $row_2$​ (the target is to eliminate (2,1), so we call it $E_{2,1}$, here $E$ means elementary or elimination)
$$
E_{2,1}=\left[\begin{matrix}1 & 0 & 0\\\\-3 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
$$
Step 2: Subtract $2\times row_2$ from $row_3$
$$
E_{3,2}=\left[\begin{matrix}1 & 0 & 0\\\\0 & 1 & 0\\\\0 & -2 & 1\end{matrix}\right]
$$
So in matrix language, the above elimination steps can be written as
$$
E_{3,2}\cdot (E_{2,1}\cdot A)=U
$$
which is extremely simple. Notice that matrix multiplication satisfies the associative law, so we can use an $E=E_{3,2}\cdot E_{2,1}$ to directly achieve the goal.

In this case we only need one type of elementary matrix: subtract $x\times row_i$ from $row_j$. But in some cases we need to do row exchanges, and we need another type of elementary matrix: the **permutation matrix**.

Under the idea of linear combinations of row vectors, it’s super-easy to construct a permutation matrix. Take the $2\times 2$ matrix as an example:
$$
\left[\begin{matrix}0 & 1\\\\1 & 0\end{matrix}\right]
\left[\begin{matrix}a & b\\\\c & d\end{matrix}\right]=
\left[\begin{matrix}c & d\\\\a & b\end{matrix}\right]
$$

> Permutation matrices that do column exchanges have the similar principle. But notice that matrices multiplied on the left are responsible for row transformation, if we want to do column transformation we need to multiply the matrix on the right, that is
>
> $$
> \begin{bmatrix}a & b\\\\c & d\end{bmatrix}\begin{bmatrix}0 & 1\\\\1 & 0\end{bmatrix}=\begin{bmatrix}b & a\\\\d & c\end{bmatrix}
> $$

---

Take the elementary matrix from the last lecture as an example to talk a little about inverse:
$$
\left[\begin{matrix}1 & 0 & 0\\\\-3 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
$$
the function of the matrix is to subtract $3\times row_1$ from $row_2$. So if there’s a matrix that can “undo” the operation, its function is to add $3\times row_1$ to $row_2$, which leads to
$$
\left[\begin{matrix}1 & 0 & 0\\\\3 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
\left[\begin{matrix}1 & 0 & 0\\\\-3 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]=
\left[\begin{matrix}1 & 0 & 0\\\\0 & 1 & 0\\\\0 & 0 & 1\end{matrix}\right]
$$
we can simply write it as $E^{-1}\cdot E=I$. The matrix on the left is the **inverse matrix** of the matrix on the right.