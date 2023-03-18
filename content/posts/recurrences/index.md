---
title: 'Homogeneous Linear Recurrences: An Algorithmic Perspective'
summary: 'How to quickly solve homogeneous linear recurrences with the aid of computers :rocket:'
categories:
- Theoretical Computer Science
date: "2023-03-18"
featured: false
math: true
---

- [Getting Started: Fibonacci](#getting-started-fibonacci)
  - [A Variation of Fibonacci](#a-variation-of-fibonacci)
- [When It Comes to Higher Orders…](#when-it-comes-to-higher-orders)
- [An “Eigendecomposition” Approach](#an-eigendecomposition-approach)
  - [Fibonacci: Review I](#fibonacci-review-i)
  - [Cayley-Hamilton Theorem](#cayley-hamilton-theorem)
  - [The Characteristic Polynomial For Homogeneous Linear Recurrence Matrices](#the-characteristic-polynomial-for-homogeneous-linear-recurrence-matrices)
  - [Magical Decomposition](#magical-decomposition)
  - [Summary \& Complexity Analysis](#summary--complexity-analysis)
- [A “Generatingfunctionology” Approach](#a-generatingfunctionology-approach)
  - [Fibonacci: Review II](#fibonacci-review-ii)
  - [Generating Function Equation for General Linear Recurrences](#generating-function-equation-for-general-linear-recurrences)
  - [An Even-Odd Separation Idea](#an-even-odd-separation-idea)
  - [Summary \& Complexity Analysis](#summary--complexity-analysis-1)
- [Conclusion \& Misc](#conclusion--misc)
- [References](#references)


The post focuses on solving homogeneous linear recurrences quickly with the aid of computers. Specifically, we are interested in figuring out the $n$-th term of sequences with the following recurrence formula:

$$
f_n=\sum_{k=1}^m c_kf_{n-k}
$$

Since the answer may be too large, we usually focus on $f'_n=f_n\mod M$. We are more interested in cases where $M$ is suitable for FFT, such as $998,244,353$, because polynomial-related techniques can be leveraged to accelerate the computation. In the whole passage, we will use $f_n$ (instead of $f'_n$) to represent the answer under modulation for simplicity.

## Getting Started: Fibonacci

Let’s start with one of the simplest (and most famous) sequences: Fibonacci.

$$
F_n=\begin{cases}0&, n=1\\\\1&, n=2\\\\F_{n-1}+F_{n-2}&, n\geq 2\end{cases}
$$

Computing $F_n$ in $O(n)$ time complexity is trivial. But when $n$ is extremely large, say, $\geq 10^9$, it’s hard for general computers to get the answer within seconds. One of the most common approaches to solving such problems is using matrices.

Matrices are good at describing linear relationships, and the recurrence formula of Fibonacci happens to be linear. Therefore we can rewrite it in matrix language:

$$
\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}\begin{pmatrix}F_{n-1}\\\\F_n\end{pmatrix}=\begin{pmatrix}0\cdot F_{n-1}+1\cdot F_n\\\\1\cdot F_{n-1}+1\cdot F_n\end{pmatrix}=\begin{pmatrix}F_n\\\\F_{n+1}\end{pmatrix}
$$

$\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}$is a good matrix for “pushing” the computation of $F_n$ by 1. Although this seems trivial, when we repeatedly use the matrix to push forward our computation, things become interesting:

$$
\begin{pmatrix}F_n\\\\F_{n+1}\end{pmatrix}=\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}\cdots\left(\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}\left(\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}\begin{pmatrix}F_0\\\\F_1\end{pmatrix}\right)\right)
$$

Thanks to the associativity of matrix multiplication, we can neatly rewrite the expression as

$$
\begin{pmatrix}F_n\\\\F_{n+1}\end{pmatrix}=\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}^n\begin{pmatrix}F_0\\\\F_1\end{pmatrix}
$$

So far, we have not simplified the computation of $F_n$, and matrix multiplications even place more burden on the computer. However, we have powerful techniques to deal with exponents: repeated squaring, i.e., denote the transformation matrix as $M$, we first compute $M^2$, then compute $M^4$ by taking the square of $M^2$, and so on. In this way, we can compute $\begin{pmatrix}0 & 1\\\\1 & 1\end{pmatrix}^n$in $O(\log n)$ complexity, and finally by applying it to the initial vector $\begin{pmatrix}F_0\\\\F_1\end{pmatrix}$we’ll get $F_n$.

If you’re an experienced math learner, you must have known several ways to directly compute the general term formula of the Fibonacci sequence and may complain about this “stupid” matrix approach. However, the general term formula containing irrational coefficients is unfriendly for numerical computations. I’m not saying that those approaches are useless - instead, in the following sections we’ll come back to the basic Fibonacci example again and again and see how advanced math gives insights into efficient algorithm design.

### A Variation of Fibonacci

Before delving deeper into the problem, let's consider a slightly varied version of the Fibonacci sequence:

$$
F_n=\begin{cases}0&, n=1\\\\1&, n=2\\\\F_{n-1}+F_{n-2}+n&, n\geq 2\end{cases}
$$

Rigorously, this is not a homogeneous linear recurrence, but we list it here because it’s a nice example of designing transformation matrices. The only difference is the additional $+n$ when $n\geq 2$, but it seems much more challenging because we cannot represent $n$ as the linear combination of $F_{n-1}$ and $F_{n-2}$. 

A general insight is that we can include more terms in the “state vector” ( in the standard Fibonacci sequence it’s $\begin{pmatrix}F_{n} & F_{n+1}\end{pmatrix}$). Apart from necessary terms in the sequence, we can add “helper terms” to help us find the linear combinations of wanted expressions. Let’s try adding $n$ into the state vector:

$$
\begin{pmatrix}F_{n-2}\\\\F_{n-1}\\\\n\end{pmatrix}\overset{which\ matrix?}{\Longrightarrow}\begin{pmatrix}F_{n-1}\\\\F_{n}\\\\n+1\end{pmatrix}
$$

Unfortunately, getting $n+1$ from $n$ is impossible because we have nobody to borrow the “1” from - but why don’t we add constant 1 into our state vector as well?

$$
\begin{pmatrix}0 & 1 & 0 & 0\\\\1 & 1 & 1 & 0\\\\0 & 0 & 1 & 1\\\\0 & 0 & 0 & 1\end{pmatrix}\begin{pmatrix}F_{n-2}\\\\F_{n-1}\\\\n\\\\1\end{pmatrix}=\begin{pmatrix}F_{n-1}\\\\F_n\\\\n+1\\\\1\end{pmatrix}
$$

Once we have the transformation matrix of a sequence, the rest is simple: do matrix multiplications by repeated squaring. To solve this Fibonacci variation, we need a matrix with rank 4, even if it's a second-order recurrence. There's no fixed procedure for devising such a matrix - in some subtle cases, a little human wisdom is necessary.

## When It Comes to Higher Orders…

The examples shown above are all constant-order recurrences. In the Fibonacci example, we claim that the time complexity is $O(\log n)$ because the influence from the rank is negligible. However, if we’re confronted with recurrences with higher orders, say $k$, the time complexity of the matrix approach should be $O(k^3\log n)$ (we don’t take into account algorithms like Strassen’s multiplication because they often come with huge constants), which is prohibitively high even if $k$ is only a few thousand.

The journey has just begun! The following sections introduce 2 directions for optimization: one is related to the characteristic polynomial of matrices, and the other leverages generating functions to tackle the issue.

## An “Eigendecomposition” Approach

### Fibonacci: Review I

A common approach to optimizing matrix powering is eigendecomposition, i.e., for a matrix $M$ with rank $n$, if $M$ has $n$ independent eigenvectors, then $M$ is diagonalizable: $M=S\Lambda S^{-1}$, where $\Lambda$ is a diagonal matrix consisting of eigenvalues and $S$ is a matrix consisting of (column) eigenvectors. With this diagonalized form, powering becomes super easy: $M^n=(S\Lambda S^{-1})^n=S\Lambda^nS^{-1}$.

Come back to the Fibonacci example, the only thing left is to compute the eigenvalues of the transformation matrix. For $2\times 2$ matrices, solving the following quadratic characteristic equation is simple:

$$
M=\begin{pmatrix}0&1\\\\1&1\end{pmatrix}\qquad\qquad
\begin{align*}f(\lambda)&=\det(M-\lambda I)=0\\\\&\Longrightarrow\lambda_{1,2}=\frac{1\pm\sqrt 5}{2},\mathbf{x}=\begin{pmatrix} \lambda\\\\1\end{pmatrix}\end{align*}
$$

Therefore,

$$
\begin{align*}\begin{pmatrix}F_n\\\\F_{n+1}\end{pmatrix}&=\begin{pmatrix}0&1\\\\1&1\end{pmatrix}^n\begin{pmatrix}F_0\\\\F_1\end{pmatrix}\\\\&=\left[\begin{pmatrix}\mathbf{x}_1&\mathbf{x}_2\end{pmatrix}\begin{pmatrix}\lambda_1 & \\\\&\lambda_2\end{pmatrix}\begin{pmatrix}\mathbf{x}_1&\mathbf{x}_2\end{pmatrix}^{-1}\right]^n\begin{pmatrix}F_0\\\\F_1\end{pmatrix}\\\\&=\begin{pmatrix}\mathbf{x}_1&\mathbf{x}_2\end{pmatrix}\begin{pmatrix}\lambda_1^n&\\\\&\lambda_2^n\end{pmatrix}\begin{pmatrix}\mathbf{x}_1&\mathbf{x}_2\end{pmatrix}^{-1}\begin{pmatrix}F_0\\\\F_1\end{pmatrix}\end{align*}
$$

### Cayley-Hamilton Theorem

Unfortunately, when it comes to transformation matrices with arbitrary rank, it’s difficult to solve high-order characteristic equations and irrational eigenvalues are hard for numerical computations. However, the general idea still works: we should find nice properties related to characteristic polynomials and try to do decomposition.

We introduce the Cayley-Hamilton theorem, which describes an interesting property of square matrices.

{{% alert note %}}
$\fbox{Theorem}$ **(Cayley-Hamilton)**

In linear algebra, the Cayley-Hamilton theorem states that every square matrix over a commutative ring satisfies its own characteristic equation, i.e., denote $f(\lambda)=\det(\lambda I-A)$ as the characteristic polynomial of $A$, then $f(A)=O$.

{{% /alert %}}

Beginners may regard this theorem as trivial by the observation that $\det(AI-A)=0$. However, this “bogus proof” is totally ungrounded because $f(A)$ is a matrix, not a scalar. Since it’s not a rigorous math post, we’ll not prove this theorem in a detailed manner from scratch. Instead, we give the following lemma without proof:

$\fbox{Lemma}$ Diagonalizable matrices are dense in the space of all matrices.

> **Proof of Cayley-Hamilton theorem**
> 
> 
> We start with diagonalizable matrices. For an arbitrary diagonalizable matrix $A$ with rank $n$, it has $n$ independent eigenvectors satisfying $A\mathbf{x_i}=\lambda_i\mathbf{x_i}$. Consider $f(A)\mathbf{x_i}$, suppose $f(\lambda)$ is a polynomial with coefficient $c_i$ for term $\lambda^i$, then
> 
> $$
> \begin{align*}f(A)\mathbf{x_i}&=\left(\sum_{k=0}^nc_kA^k\right)\mathbf{x_i}=\sum_{k=0}^nc_k(A^k\mathbf{x_i})\\\\&=\sum_{k=0}^nc_k(\lambda_i^k\mathbf{x_i})\\\\&=\left(\sum_{k=0}^nc_k\lambda_i^k\right)\mathbf{x_i}\\\\&=f(\lambda_i)\mathbf{x_i}=\mathbf{0}\end{align*}
> $$
> 
> Therefore, for each eigenvector $\mathbf{x_i}$, $f(A)\mathbf{x_i}=\mathbf{0}$. By combining eigenvectors to a matrix $S=\begin{pmatrix}\mathbf{x_1}&\cdots&\mathbf{x_n}\end{pmatrix}$, we have $f(A)S=O$. Since $S$ is a full-rank matrix, the only possibility is that $f(A)=O$.
> 
> Now let’s move to non-diagonalizable matrices, denoted as $D$. According to the density of diagonalizable matrices in the whole matrix space, there exists a matrix $H$ such that a cluster of matrices $D_t=D+tH$, $t\in (-1, 0)\cup (0, 1)$, are diagonalizable. Since $D_t$ is continuous on $t$, $f_{D_t}(D_t)$ is also continuous on $t$, therefore
> 
> $$
> f(D)=\lim_{t\to 0}f_{D_t}(D_t)=O
> $$
> 

### The Characteristic Polynomial For Homogeneous Linear Recurrence Matrices

For homogeneous linear recurrence transformation matrices, characteristic polynomial $f$ is easy to compute: according to the recurrence formula, we have

$$
A=\begin{pmatrix}0& 1\\\\& & 1\\\\& & & \ddots\\\\& & & & 1\\\\c_1 & c_2 & c_3 &  \cdots & c_k\end{pmatrix}
$$

Then

$$
f(\lambda)=\det(\lambda I-A)=\begin{vmatrix}\lambda & -1\\\\& \lambda & -1\\\\& & \ddots& \ddots\\\\& & & \lambda & -1\\\\-c_1 & -c_2 & -c_3&\cdots&\lambda-c_k\end{vmatrix}
$$

Expanding the determinant by the last row, we’ll observe that every minor is easy to compute: $A_{k,i}=(-1)^{k-i}\lambda^{i-1}$. Therefore

$$
\begin{align*}f(\lambda)&=(-1)^{2k}(\lambda-c_k)\lambda^{k-1}+\sum_{i=1}^{k-1}(-1)^{k+i}\cdot ((-1)^{k-i}\lambda^{i-1})\cdot (-c_i)\\\\&=\lambda^k-\sum_{i=1}^kc_i\lambda^{i-1}\end{align*}
$$

### Magical Decomposition

It’s still unclear how Cayley-Hamilton theorem can help in optimization. The following approach was proposed by Fiduccia in 1985 [2]. For an arbitrary transformation matrix $A$ with rank $k$, consider the following decomposition:

$$
x^n=f(x)g(x)+r(x)
$$

where $f$ is the characteristic polynomial of $A$ and  $g(x)/r(x)$ can be computed through polynomial division/modulation. In fact, we are more interested in $r(x)$ because when we let $x=A$:

$$
A^n=f(A)g(A)+r(A)\overset{\text{Cayley-Hamilton}}{=}r(A)
$$

It seems that we haven’t made solid progress because $r$ is a $k$-order polynomial and computing $r(A)$ by brute force requires $O(k^4)$ effort. However, suppose $\displaystyle r(x)=\sum_{i=0}^{k-1}c_ix^i$, actually we’re able to compute $F_n$ in $O(k)$ thanks to the following deduction:

$$
\begin{align*}\begin{pmatrix}F_n\\\\F_{n+1}\\\\\vdots\\\\F_{n+k-1}\end{pmatrix}&=A^n\begin{pmatrix}F_0\\\\F_1\\\\\vdots\\\\F_{k-1}\end{pmatrix}=R(A)\begin{pmatrix}F_0\\\\F_1\\\\\vdots\\\\F_{k-1}\end{pmatrix}\\\\&=\left(\sum_{i=0}^{k-1}c_iA^i\right)\begin{pmatrix}F_0\\\\F_1\\\\\vdots\\\\F_{k-1}\end{pmatrix}\\\\&=\sum_{i=0}^{k-1}c_i\left(A^i\begin{pmatrix}F_0\\\\F_1\\\\\vdots\\\\F_{k-1}\end{pmatrix}\right)\\\\&=\sum_{i=0}^{k-1}c_i\begin{pmatrix}F_i\\\\F_{i+1}\\\\\vdots\\\\F_{i+k-1}\end{pmatrix}\end{align*}
$$

Therefore

$$
F_n=\sum_{i=0}^{k-1}c_iF_i
$$

### Summary & Complexity Analysis

Let’s have a quick summary of the whole procedure and analyze the complexity:

1. Construct a transformation matrix $A$ according to the recurrence formula.
2. Compute the characteristic polynomial $f_A$.
3. Compute $r_A(x)=x^n\text{ mod }f_A$.
4. Compute $F_n$ according to the coefficients in $r_A$.

The main complexity lies in the third step. Since $n$ is extremely large, it’s impossible to directly do polynomial modulation. Instead, we can leverage repeated squaring:

$$
x\Rightarrow [x^2]=(x)^2 \text{ mod }f_A\Rightarrow [x^4]=[x^2]^2\text{ mod }f_A\Rightarrow \cdots \Rightarrow x^n\text{ mod }f_A
$$

In this way, we need to conduct $O(\log n)$ times polynomial multiplication and modulation, which gives a $O(k^2\log n)$ complexity. If the problem is considered in a ring that supports Fast Fourier Transformation (typically an NTT ring), we can further optimize it to $O(k\log k\log n)$.

## A “Generatingfunctionology” Approach

### Fibonacci: Review II

Generating function is always a powerful weapon for tackling sequence problems. Consider

$$
G(x)=\sum_{n=0}^\infty F_nx^n
$$

Since $F_n=F_{n-1}+F_{n-2}$ when $n\geq 2$, we can get

$$
G(x)=F_0+F_1x+xG(x)+x^2G(x)=x+(x+x^2)G(x)
$$

Therefore $\displaystyle G(x)=\frac{x}{1-x-x^2}=\frac{1}{\sqrt 5}\left(\frac{1}{1-\phi x}-\frac{1}{1-\hat\phi x}\right)$, where $\phi=\displaystyle \frac{1+\sqrt 5}{2}$ and $\displaystyle \hat\phi=\frac{1-\sqrt 5}{2}$. Expand it to Taylor series, we have

$$
\begin{align*}G(x)&=\frac{1}{\sqrt 5}\left(\sum_{n\geq 0}(\phi x)^n-\sum_{n\geq 0}(\hat\phi x)^n\right)\\\\&=\sum_{n\geq 0}\left(\frac{1}{\sqrt 5}(\phi^n-\hat\phi^n)\right)x^n\end{align*}
$$

Thus $\displaystyle F_n=\frac{1}{\sqrt 5}(\phi^n-\hat\phi^n)$.

### Generating Function Equation for General Linear Recurrences

However, when it comes to recurrences with arbitrary order, techniques like Taylor expansion will lose its power for computers. Anyway, let’s figure out the equation first. Put the coefficients of linear recurrence into a generation function, denoted as $G(x)$, i.e.,

$$
G(x)=\sum_{i=1}^kc_ix^i
$$

we have

$$
F(x)=F(x)G(x)+R(x)
$$

Where $R(x)$ is a polynomial for correcting the first few terms in the sequence and can be obtained by computing $F'(x)-F'(x)G(x)$, where $F'(x)$ is the first $k$ terms of $F(x)$.

Therefore

$$
F(x)=\frac{R(x)}{1-G(x)}
$$

### An Even-Odd Separation Idea

Since $n$ is extremely huge, computing the polynomial inverse of $1-G(x)$ and then computing  $[x^n]F(x)$ through polynomial multiplication is unreasonable. We need a quick way to compute

$$
[x^n]F(x)=[x^n]\frac{P(x)}{Q(x)}
$$

where $P(x)$ and $Q(x)$ stands for $R(x)$ and $1-G(x)$ in this case.

Consider the following transformation:

$$
F(x)=\frac{P(x)}{Q(x)}=\frac{P(x)Q(-x)}{Q(x)Q(-x)}
$$

$Q(x)Q(-x)$ is an even function. Let $U(x^2)=Q(x)Q(-x)$, and divide $P(x)Q(-x)$ into two parts according to the parity of $x^k$, we have

$$
F(x)=\frac{xOdd(x^2)}{U(x^2)}+\frac{Even(x^2)}{U(x^2)}
$$

Thanks to the fact that $U(x^2)$ only contains even powers, $F(x)$ is successfully separated as an odd part and an even part, and we can choose the part containing $x^n$ (according to the parity of $n$) to continue our computation.

### Summary & Complexity Analysis

Let’s have a quick summary of the whole procedure and analyze the complexity:

1. Compute $R(x)$ and $G(x)$.
2. Compute $P(x)Q(-x)$ and $Q(x)Q(-x)$ respectively. Choose the part with the same parity as $n$.
3. Repeatedly conduct step 2 until the polynomial becomes a constant one.

Step 2 contains polynomial multiplications, which requires $O(k^2)$/$O(k\log k)$ depending on whether FFT is supported. Every time we conduct step 2, the formal variable of the generating functions grow from $x$ to $x^2$. Therefore the power grows exponentially and we can terminate within $O(\log n)$ iterations. So the overall complexity is $O(k^2\log n)$/$O(k\log k\log n)$.

It’s worth noting that although the “generatingfunctionology” approach has the same time complexity as the previous one, it requires simpler polynomial techniques (only multiplication). The paper [3] argues that this approach improves the result of Fiduccia on constant factors.

## Conclusion & Misc

The post introduces several kinds of techniques for tackling homogeneous linear recurrence problems. Matrix multiplication with repeated squaring optimizes the complexity from the most trivial $O(kn)$ to $O(k^3\log n)$, and the following 2 approaches leverage linear algebra/generating function to further improve it to $O(k^2\log n)$/$O(k\log k\log n)$. Readers can refer to the papers for deeper understanding.

I learned most of the techniques mentioned above when I was taking part in competitive programming contests during high school. There was no detailed tutorial on the Internet at that time and the knowledge are spread through word of mouth. I would be most happy if the post can bring the wisdom of TCS researchers to more people in the world. 

## References

- [1]  [Cayley-Hamilton Theorem](https://en.wikipedia.org/wiki/Cayley%E2%80%93Hamilton_theorem). *Wikipedia*.
- [2] C. M. Fiduccia. [An efficient formula for linear recurrences](https://doi.org/10.1137/0214007). *SIAM Journal on Computing*,
14(1):106–112, 1985.
- [3] Bostan, A., & Mori, R. (2020). [A Simple and Fast Algorithm for Computing the N-th Term of a Linearly Recurrent Sequence](https://arxiv.org/abs/2008.08822). *ArXiv.*