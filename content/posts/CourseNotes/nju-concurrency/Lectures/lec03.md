---
title: "Lecture 03: Basic Operational Semantics for Concurrency"
linktitle: 'Lecture 03'
type: docs
date: '2022-09-20'
draft: false
featured: false

math: true

menu:
    nju-concurrency-lectures:
        parent: Contents
        weight: 3
---

- [A Simple Concurrent Programming Language](#a-simple-concurrent-programming-language)
- [Basic Setup](#basic-setup)
  - [Thread Subsystem](#thread-subsystem)
  - [Storage Subsystem](#storage-subsystem)
  - [Linking the Two](#linking-the-two)
- [The Thread Subsystem](#the-thread-subsystem)
  - [Thread Local Steps](#thread-local-steps)
  - [Lifting to Concurrent Programs](#lifting-to-concurrent-programs)
- [SC Storage subsystem](#sc-storage-subsystem)
  - [Linking the Thread and Storage Subsystems](#linking-the-thread-and-storage-subsystems)
- [x86's TSO Storage Subsystem:](#x86s-tso-storage-subsystem)
  - [Linking The Thread and Storage Subsystem](#linking-the-thread-and-storage-subsystem)
- [Exercise: PSO Storage System](#exercise-pso-storage-system)

## A Simple Concurrent Programming Language

**Basic domains:**

* Registers: $r\in \text{Reg}$. 
* Memory locations: $x\in \text{Loc}$. 
* Values (including 0): $v\in \text{Val}$.
* Thread identifiers: $i\in \text{Tid}=\\{1,\cdots,N\\}$.

**Expressions and commands** (here all commands are atomic):

```plaintext
e ::= r | v | e +(-*/) e | ...

c ::= skip | if e then c else c | while e do c |
      c ; c (sequential) | r := e | r := x (read) | x := e (write)
      r := FAA(x, e) | r := CAS(x, e, e) | fence
```

* FAA means "fetch-and-add", `r = FAA(x, e)` will do

    ```c++
    r = x;
    x += e;
    ```

    atomically.

* CAS means "compare-and-set/swap", `r = CAS(x, er, ew)` will do

    ```c++
    r = x;
    if (r == er) {
        x = ew;
        r = 1;
    }
    else
        r = 0;
    ```

    atomically.

* `fence` only takes effect under WMM. It forces previous writes to be propagated into memory.

**Programs:** $P:Tid\to Cmd$, written as $P=c_1||c_2||\cdots ||c_N$.

## Basic Setup

Different memory models usually share the same thread behaviors while have different implementation on the read/write side. Therefore we decompose the system into thread subsystem and storage subsystem and set an "virtual" interface between them.

### Thread Subsystem

* thread local step: $(c,s)\overset{l}{\to}(c',s')$, 
* Lift them to program steps: $(P,S)\overset{i,l}{\to}(P',S')$.

### Storage Subsystem 

* It's memory-model-dependent.

* Describe the effect of memory accesses and fences.

* $M\overset{i:l}{\to}M'$.

### Linking the Two

Either the thread or the storage subsystem make an internel step, i.e., $\epsilon$, or they make matching $i:l$ steps. The transformation is denoted as $P,S,M\Rightarrow P',S',M'$.

## The Thread Subsystem 

### Thread Local Steps

**Store:** $s:\text{Reg}\to \text{Val}$ (initially $s_0\triangleq \lambda r.0$).

**State:** $(c,s)\in \text{Command}\times \text{Store}$

**Transitions:**

$\mathbf{skip}$:
$$
\frac{}{\mathbf{skip};c,s\overset{\epsilon}{\to}c,s}
$$

Sequential:

$$
\frac{c_1,s\overset{l}{\to}c_1',s'}{c_1;c_2,s\overset{l}{\to}c_1',c_2,s'}
$$

Register operations:
$$
\frac{s'=s[r\mapsto s(e)]}{\text{r:=e},s\overset{\epsilon}{\to}\mathbf{skip},s'}
$$
Memory read:
$$
\frac{l=R(x, v)}{\text{r:=x},s\overset{l}{\to}\mathbf{skip},s[r\to v]}
$$
Here $l=R(x, v)$ means that we read from the memory and $M[x] = v$.

Memory write:
$$
\frac{l=W(x, s(e))}{\text{x:=e,s}\overset{l}{\to}\mathbf{skip},s}
$$
$\mathbf{if}$-$\mathbf{then}$-$\mathbf{else}$ branch:
$$
\frac{s(e)\neq 0}{\mathbf{if}\space e\space \mathbf{then}\space c_1\space \mathbf{else}\space c_2,s\overset{\epsilon}{\to}c_1,s}
$$

$$
\frac{s(e)= 0}{\mathbf{if}\space e\space \mathbf{then}\space c_1\space \mathbf{else}\space c_2,s\overset{\epsilon}{\to}c_2,s}
$$

$\mathbf{while}$ loop:
$$
\frac{}{\mathbf{while}\space e\space \mathbf{do}\space c,s\overset{\epsilon}{\to}\mathbf{if}\space e\space\mathbf{then}\space (c;\mathbf{while}\space e\space \mathbf{do}\space c)\space \mathbf{else}\space \mathbf{skip},s}
$$
Fetch-and-add:
$$
\frac{l=U(x, v, v + s(e))}{\text{r:=}\mathbf{FAA}(x,e),s\overset{l}{\to}\mathbf{skip},s[r\to v]}
$$
Here we define a new way to interact with memory: `U` (update).  `U(x, vr, vw)` means that we fetch $v_r$ from $M[x]$ and write $v_w$ to $M[x]$.

Compare-and-swap:
$$
\frac{l=R(x, v)\quad v\neq s(e_r)}{\text{r:=}\mathbf{CAS}(x, e_r,e_w),s\overset{l}{\to}\mathbf{skip},s[r\to 0]}
$$

$$
\frac{l=U(x, s(e_r), s(e_w))}{\text{r:=}\mathbf{CAS}(x, e_r,e_w),s\overset{l}{\to}\mathbf{skip},s[r\to 1]}
$$

It's worth noticing that when compare-and-swap fails, it acts as a normal read operation ("R" instead of "U")

Fence:
$$
\frac{}{\mathbf{fence},s\overset{F}{\to}\mathbf{skip},s}
$$
For WMM, when the memory receives "F", it will do particular operations.

### Lifting to Concurrent Programs

**State:** $(P,S)\in\text{Program}\times (\text{Tid}\to \text{Store})$. (initially $(P,S_0)$, where $S_0\triangleq \lambda i.s_0$)

**Transitions:**
$$
\frac{P(i),S(i)\overset{l}{\to}c,s}{P,S\overset{i:l}{\mapsto}P[i\to c],S[i\mapsto s]}
$$

## SC Storage subsystem

**Machine state:** $M:\text{Loc}\to\text{Val}$ (initially $M_0\triangleq \lambda x.0$).

**Transitions:**
$$
\frac{l=W(x, v)}{M\overset{i:l}{\to}M[x\mapsto v]}
$$

$$
\frac{l=R(x, v)\quad M[x]=v}{M\overset{i:l}{\to}M}
$$

$$
\frac{l=U(x,v_r,v_w)\quad M[x]=v_r}{M\overset{i:l}{\to}M[x\mapsto v_w]}
$$

$$
\frac{l=F}{M\overset{i:l}{\to}M}
$$

(`fence` doesn't take effect in SC memory model.)

### Linking the Thread and Storage Subsystems

Silent:
$$
\frac{P,S\overset{i:\epsilon}{\to}P', S'}{P,S,M\Rightarrow P',S',M}
$$
Non-silent:
$$
\frac{P,S\overset{i:l}{\to}P',S'\quad M\overset{i:l}{\to}M'}{P,S,M\Rightarrow P',S',M'}
$$
An outcome $O:\text{Tid}\to\text{Store}$ is allowed for a program $P$ under SC if there exists $M$ such that
$$
P,S_0,M_0\Rightarrow^* \mathbf{skip}||...||\mathbf{skip},O,M.
$$

## x86's TSO Storage Subsystem:

**Machine states:**

* A memory $M:\text{Loc}\to\text{Val}$
* A function $B:\text{Tid}\to (\text{Loc},\text{Val})^*$ assigning a **store buffer** to every thread. Here a pair $(x, v)$ stands for a write. The most recent write appears at the left most position of the write sequence.

(Initially, $M_0\triangleq \lambda x.0$,$B_0=\lambda i.\epsilon$).

**Transitions:**

Write (only to the thread-local buffer):
$$
\frac{l=W(x, v)}{M,B\overset{i:l}{\to}M,B[i\mapsto (x,v)\cdot B(i)]}
$$
Propagate (flush the least recent operation in the buffer to the memory):
$$
\frac{B(i)=b\cdot (x,v)}{M,B\overset{i:\epsilon}{\to}M[x\mapsto v],B[i\mapsto b]}
$$
Read
$$
\frac{l=R(x,v)\quad B(i)=(x_n,v_n)\cdot\ldots\cdot(x_1,v_1)\quad M[x_1\mapsto v_1]\cdots[x_n\mapsto v_n] (x)=v}{M,B\overset{i:l}{\to}M,B}
$$
Notice that during a read, the buffer won't be flushed into the memory.

RMW (read-modify-write):
$$
\frac{l=U(x,v_r,v_w)\quad B(i)=\epsilon \quad M(x)=v_r}{M,B\overset{i:l}{\to}M[x\mapsto v_w],B}
$$
The condition $B(i)=\epsilon$ indicates that RMW will automatically synchronize the state. Notice that in $\mathbf{CAS}$, if the comparison failed, $\mathbf{CAS}$ would act as a normal "read" operation and in TSO no synchronization would be conducted.

Fence:
$$
\frac{l=F\quad B(i)=\epsilon}{M,B\overset{i:l}{\to}M,B}
$$
Fence is used as forced synchronization.

### Linking The Thread and Storage Subsystem

Silent thread:
$$
\frac{P,S\overset{i:\epsilon}{\to}P',S'}{P,S,M,B\Rightarrow P,S',M,B}
$$
Silent storage:
$$
\frac{M,B\overset{i:\epsilon}{\to}M',B'}{P,S,M,B\Rightarrow P,S,M',B'}
$$
Non silent:
$$
\frac{P,S\overset{i:l}{\to}P',S'\qquad M,B\overset{i:l}{\to}M',B'}{P,S,M,B\Rightarrow P',S',M',B'}
$$
The definition of allowed outcome is similar to that of SC.

## Exercise: PSO Storage System

*Partial Store Ordering (PSO)* is a WMM similar to TSO, but it does not guarantee that stores to different locations propagate to the main memory in the order they were issued. In particular, it allows the following weak behavior:

```c++
Initially, x = y = 0;
x = 1;  ||  a = y; // 1
y = 1;  ||  b = x; // 0;
```

(1) Operational semantics for PSO:

We only need to redefine the propagate rule in TSO:
$$
\frac{B(i)=b'\cdot(x,v)\cdot b\qquad b=(x_t,v_t)\cdot\ldots\cdot(x_1,v_1),\forall 1\leq i\leq t,x_i\neq x}{M,B\overset{i:\epsilon}{\to}M[x\mapsto v],B[i\mapsto b'\cdot b]}
$$
(2) store-store fence:

In the thread subsystem:
$$
\frac{}{\mathbf{ssfence};c,s\overset{SSF}{\to}\mathbf{skip};c,s}
$$
In the storage subsystem:

SSF rule: add a special mark into the buffer array:
$$
\frac{}{M,B\overset{SSF}{\to}M,B[i\mapsto F\cdot B(i)]}
$$
Propagation rule needn't be changed because our conditions implicitly require that there's no store-store fence before the target operation. But we need an additional rule to move away "F" marks:
$$
\frac{B(i)=b\cdot F}{M,B\overset{i:\epsilon}{\to}M',B[i\mapsto b]}
$$
(3) Show that programs containing store-store fences between every two writes have the same outcomes under TSO and PSO:

TSO and PSO differ only in their propagation rules. If there is a store-store fence between every two writes, it's easy to prove that at any point, the buffer of arbitrary thread should be in the form of $(x_n,v_n)\cdot F\cdot \ldots \cdot F\cdot (x_2,v_2)\cdot F\cdot (x_1,v_1)$. Therefore at any point the propagation rule in PSO can only choose the least recent operation in the buffer to propagate, which makes it completely the same as TSO's propagation rule.