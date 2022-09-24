---
title: "Lecture 02: Mathematical Background"
linktitle: '02: 数学基础'
type: docs
date: '2022-09-14'
draft: false
featured: false

math: true

menu:
    nju-formalsemantics-lectures:
        parent: Contents
        weight: 2
---

- [Sets](#sets)
  - [Generalized Unions of Sets](#generalized-unions-of-sets)
  - [Generalized Intersections of Sets](#generalized-intersections-of-sets)
- [Relations](#relations)
  - [Basic notations](#basic-notations)
  - [Equivalence Relations](#equivalence-relations)
- [Functions](#functions)
  - [Variation](#variation)
  - [Function Types](#function-types)
  - [Currying](#currying)
- [Products](#products)
  - [Tuples as Functions](#tuples-as-functions)
  - [Generalized Products](#generalized-products)
  - [Exponentiation](#exponentiation)
- [Sums](#sums)

## Sets

### Generalized Unions of Sets

$$
\begin{align}
\bigcup S &\triangleq \\{x\space |\space \exists \space T\in S. x\in T\\}\\\\
\bigcup_{i\in I} S(i) &\triangleq \bigcup\\{S(i) \space|\space i\in I\\}\\\\
\bigcup_{i=m}^n S(i) &\triangleq \bigcup_{i\in [m,n]} S(i)
\end{align}
$$

Note: in the first definition, $S$ is a a set of sets.

### Generalized Intersections of Sets

$$
\begin{align}
\bigcap S &\triangleq \\{x\space|\space \forall\space T\in S. x\in T\\}\\\\
\bigcap_{i\in I} S(i) &\triangleq \bigcup\\{S(i)\space |\space i\in I\\}\\\\
\bigcap_{i=m}^n S(i) &\triangleq \bigcup_{i\in [m,n]} S(i)
\end{align}
$$

Let's consider $\emptyset$. For union:
$$
\begin{align}
\bigcup \emptyset &= \\{x\space|\space \exists\space T\in \emptyset. x\in T\\}\\\\
&= \\{x\space|\space \exists\space T. T\in \emptyset\space \wedge x\in T\\}\\\\
&=\emptyset
\end{align}
$$
For intersection:
$$
\begin{align}
\bigcap \emptyset &= \\{x\space|\space \forall\space T\in \emptyset.x\in T\\}\\\\
&=\\{x\space|\space \forall\space T.T\in \emptyset\to x\in T\\}\\\\
&=\\{x\space|\space \forall\space T.\text{false}\to x\in T\\}\\\\
\end{align}
$$
This results in "a set of everything", which contradicts the Russel paradox. Therefore $\bigcap \emptyset$ is meaningless.

> **Russel Paradox**
>
> If there exists "a set of everything", denoted as $A$. Consider
> $$
> B=\\{x\in A\space |\space x\notin x\\}
> $$
> Since $A$ is a set of everything, $B\in A$. The question is whether $B\in B$.
>
> * If $B\in B$, then according to the definition of $B$, $B\notin B$ holds.
> * If $B\notin B$, then according to the definition of $B$, $B\in B$ holds.

## Relations

$$
A\times B\triangleq \\{(x, y)\space|\space x\in A \text{ and }y\in B\\}
$$

Here $(x, y)$ is called a pair. We denote $\pi_0(x, y)=x,\pi_1(x, y)=y$.

$\rho$ is a relation from $A$ to $B$ if $\rho \subseteq A\times B$, or equivalently, $\rho \in \mathcal P(A\times B)$

### Basic notations

* Identity: $\text{Id}_s\triangleq \\{(x, y)\space |\space x\in S\\}$.
* Domain: $dom(\rho)\triangleq \\{x\space |\space \exists\space y. (x,y)\in \rho\\}$.
* Range: $ran(\rho)\triangleq \\{y\space |\space \exists\space x. (x, y)\in\rho\\}$.
* Composition: $\rho'\circ\rho\triangleq \\{(x,z)\space |\space \exists\space y.(x,y)\in \rho \space\wedge\space (y,z)\in\rho'\\}$.
* Inverse: $\rho^{-1}\triangleq \\{(x, y)\space |\space (y,x)\in \rho\\}$

### Equivalence Relations

$\rho$ is an equivalence relation on $S$ if it's reflexive, symmetric and transitive.

* Reflexivity: $\text{Id}_s\subseteq \rho$.
* Symmetry: $\rho^{-1}\subseteq \rho$.
* Transitivity: $\rho \circ\rho \subseteq \rho$.

## Functions

A relation $\rho$ is a function if for all $x,y,y'$, $(x,y)\in \rho$ and $(x,y')\in\rho$ imply $y=y'$.

Empty set is a special function (check the definition).

Functions can be denoted by Typed Lambda Expressions: $\lambda x\in S. E$ denotes the function $f$ with domain $S$ such that $f(x)=E$ for all $x\in S$.

### Variation

Variation of a function at a single argument:
$$
f\\{x\rightsquigarrow n\\}\triangleq \lambda z.
\begin{cases}
f\space z&, z\neq x\\\\
n&,z=x
\end{cases}
$$
Note that $x$ does not have to be in $dom(f)$.

### Function Types

We use $A\to B$ to represent the set of all functions from $A$ to $B$.

 $\to$ is right associative, i.e., $A\to B\to C=A\to(B\to C)$.

Example: if $f\in A\to B\to C$, then for any $a\in A$ and $b\in B$  $f\space a\space b=(f(a))b\in C$ (here $f(a)$ is another function).

### Currying

$f\in A_1\times A_2\times \cdots \times A_n\to B$, $f(a_1, a_2, \cdots, a_n)=b$

Currying: $g\in A_1\to A_2\to \cdots A_n$. $g\space a_1\space a_2\cdots a_n=b$.

## Products

Product:
$$
S_0\times S_1\times S_{n-1}\triangleq \\{(x_0,\cdots, x_{n-1}\space|\space \forall\space i=1,\cdots ,n.x_i\in S_i\\}
$$
$\pi_i(x_0,\cdots, x_{n-1})=x_i$.

### Tuples as Functions

$(x_0,x_1,\cdots, x_{n-1})$ can be viewed as a function:
$$
\lambda i\in \mathbf{n}.\begin{cases}x_0&,i=0\\\\x_1&,i=1\\\\&\cdots\\\\x_{n-1}&,i={n-1}\end{cases}
$$
where $\mathbf{n}=\\{0,1,\cdots n-1\\}$.

Then
$$
S_0\times \cdots \times S_{n-1}\triangleq \\{f\space|\space dom(f)=\mathbf{n}, \forall\space i\in \mathbf{n}, f\space i\in S_i\\}
$$

### Generalized Products

$$
\prod_{i\in I}S(i)\triangleq \\{f\space |\space dom(f)=I, \forall \space i\in I,f\space i\in S(i)\\}
$$

Let $\theta$ be a function from a set of indices to a set of sets, i.e., $\theta$ is an indexed family of sets. Then
$$
\Pi\theta\triangleq \\{f\space |\space dom(f)=dom(\theta), \forall\space i\in dom(\theta).f\space i\in \theta\space i\\}
$$
Some interesting corner cases:

* $\Pi \emptyset=\\{\emptyset\\}$.
* If there exists $i\in dom(\theta)$, $\theta\space i=\emptyset$, then $\Pi\theta=\emptyset$.

### Exponentiation

We denote $S^T$ for $\displaystyle{\prod_{x\in T} S}$ if $S$ is independent of $x$. An interesting fact is that
$$
\begin{align}
S^T&=\prod_{x\in T}S=\Pi\lambda x\in T. S\\\\
&=\\{f\space|\space dom(f)=T,\forall\space x\in T,f\space x\in S\\}\\\\
&=(T\to S)
\end{align}
$$
We always use $\mathbf{2}^S$ for power set $\mathcal P(S)$.  That's because $\mathbf{2}^S=(S\to 2)$. For any subset $T$ of $S$, we can define
$$
f = \lambda x\in S. \begin{cases}1&, x\in T\\\\0&,x\notin T\end{cases}
$$
Then $f\in (S\to 2)$. For any $f\in (S\to \mathbf{2})$, we can also construct a subset of $S$.

## Sums

$$
S_0+S_1+\cdots+S_n\triangleq \\{(i,x)\space|\space i\in \mathbf{n}, x\in S_i\\}
$$

Generalized form:
$$
\sum_{i\in I}S(i)\triangleq\\{(i,x)\space |\space i\in I, x\in S(i)\\}
$$

$$
\Sigma\theta\triangleq \\{(i,x)\space |\space i\in dom(\theta),x\in \theta\space i\\}
$$

We can prove that $\displaystyle{\sum_{x\in T}S}=T\times S$:
$$
\begin{align}
\sum_{x\in T}S=\\{(x,y)\space|\space x\in T,y\in S\\}=T\times S
\end{align}
$$