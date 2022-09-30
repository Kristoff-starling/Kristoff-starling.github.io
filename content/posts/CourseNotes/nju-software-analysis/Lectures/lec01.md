---
title: "Lecture 01: Introduction"
linktitle: 'Lecture 01'
date: 2022-09-15
type: docs
draft: false

math: true
diagram: true

menu:
    nju-static-lecture:
        parent: Contents
        weight: 1
---

- [PL and Static Analysis](#pl-and-static-analysis)
- [Why We Learn Static Analysis](#why-we-learn-static-analysis)
- [What is Static Analysis](#what-is-static-analysis)
- [Static Analysis Features and Examples](#static-analysis-features-and-examples)
  - [Static Analysis: Bird's Eye View](#static-analysis-birds-eye-view)
  - [Static Analysis: An Example](#static-analysis-an-example)
    - [Abstraction](#abstraction)
    - [Over-approximation](#over-approximation)

## PL and Static Analysis

Programming Languages:

* Theory: language design, type system, semantics and logics. etc.
* Environment: compilers, runtime system etc.
* Application: ensure the soundness, performance, security of projects => program analysis/verification/synthesis.

Background: in recent decades, the language cores had few changes, but the programs became significantly larger and more complicated.

> **PL Categories**
>
> * Imperative languages (e.g. Java, C++): fine control on program states (hardware)
> * Functional languages (e.g. Haskell, OCaml, scheme): reduce side effects (states of programs), but less control on states
> * Declarative languages (e.g. SQL, prolog) abstraction on computing process, describe targets of programs.

Challenge: How to ensure the reliability, security and other promises of large-scale and complex programs?

## Why We Learn Static Analysis

Static Analysis (static means at compile time, before execution) are helpful in

* Program Reliability: null pointer dereferences, memory leak etc.
* Program Security: private information leak, injection attack etc.
* Compiler Optimization: dead code elimination, code motion etc.
* Program Understanding: IDE call hierarchy, type indication etc.

## What is Static Analysis

Static analysis analyzes a program $P$ to reason about its behaviors and determines whether it satisfies some properties **before running** $P$.

* Can two pointers point to the same memory location?
* Does $P$ dereference any null pointers?
* Will certain `assert()` statements in $P$ fail?
* ……

Unfortunately, by Rice's Theorem, there is no such approach to determine whether $P$ satisfies such non-trivial properties, i.e., giving exact answer: Yes or No.

> **Rice Theorem**
>
> Any non-trivial property of the behavior of programs in a r.e.(recursively enumerable, recognizable by a Turning-machine) language is undecidable.
>
> A property is trivial if either it's not satisfied by any language or it's satisfied by all languages. (Non-trivial properties $\approx$ properties related with run-time behaviors of programs)

“Perfect", i.e., sound and complete, static analysis does not exist.

> **Soundness and Completeness**
>
> Soundness: over-approximation, the result is a superset of the truth.
>
> Completeness: under-approximation, the result is a subset of the truth.

Useful static analysis should compromise. If we compromise soundness, we introduce false negatives; if we compromise completeness, we introduce false positives. Mostly static analysis techniques compromise completeness, i.e. sound but not fully precise static analysis.

Soundness is critical to a collection of important applications. For example:

```c++
B b = new B();    |    C c = new C();
a.fld = b;        |    a.fld = c;

           fld = (B) a.fld;   // <- is the casting safe?
```

(This is a control flow graph, not concurrent threads.) If the analysis is unsound, e.g. only contains the behavior of the first snippet, we may come to the false conclusion that the casting is safe.

## Static Analysis Features and Examples

### Static Analysis: Bird's Eye View

```c++
if (input) x = 1; else x = 0;
// x = ?
```

Two analysis results:

* When `input` is true, x = 1; when `input` is false, x = 0. - Sound, precise, expensive.
* x = 1 or x = 0. - Sound imprecise, cheap.

The goal of static analysis is to ensure soundness while making good trade-offs between analysis precision and analysis speed.

Two words to conclude: **abstraction** & **over-approximation**.

### Static Analysis: An Example

Target: determine the sign (+/-/0) of all the variables of a given program (useful in divided-by-0 detection, negative indices detection...).

#### Abstraction

Under this specific goal we don't care about the exact value (e.g. 10 or 20), we only care about the sign. The abstraction construct a mapping from concrete domain to abstract domain: ($+,-,0,T,\top,\bot$) ($\top$ stands for unknown, $\bot$ stands for undefined). 

#### Over-approximation

* Transfer functions define how to evaluate different program statements on abstract values.

    * e.g. `+` +`+` =`+`, `+` +`-` =`T`.

    * ```c++
        x = 10;     +
        y = -1;     -
        z = 0;      0
        a = x + y;  unknown
        b = z / y;  0
        c = a / b;  undefined -> divided by zero
        p = arr[y]; undefined -> negative index
        q = arr[a]; undefined -> negative index
        ```

        `c` and `p` shows the usefulness of static analysis. `q` shows that static analysis may lead to false positives.

* Control flow

    ```c++
    x = 1;
    if (input) y = 10; else y = -1;
    z = x + y;
    ```

    ```mermaid
    graph TD
    S1(x=1) --> |y=+|S2(y=10)
    S1 --> |y=-|S3(y=-1)
    S2 --> S4("z=x+y<br>(merge: y=unknown)")
    S3 --> S4
    ```

    As it's impossible to enumerate all paths in practice, flow merging is taken for granted in most static analysis.