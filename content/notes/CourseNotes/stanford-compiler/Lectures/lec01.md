---
title: "Stanford-CS143 Lecture 01: Introduction"
linktitle: 'Lecture 01'
date: 2022-07-01
type: docs
draft: false
summary: A brief introduction to interpreters and compilers + The economy of programming languages

math: true

menu:
    stanford-compiler-lectures:
        parent: Contents
        weight: 1
---

Interpreters: it takes in program and data and and immediately generates output (an on-line process).

Compilers: it takes in program and generates an executable (assembly/bytecode). The executable takes in data and generates output (an off-line process).

## The Structure of Compilers

* Lexical Analysis: divide program text into "words" or "tokens".

    ```basic
    if x == y then z = 1; else z = 2;
    ```

    tokens: `if` `x`  `==` `z` `then` `z` `=` `1` `;` `else` `z` `=` `2` `;`

* Parsing: understand the sentence structure.

    ```basic
    if x == y then z = 1; else z = 2;
    ```

    ```
    if-then-else-+-predicate---relation-+-'x'
                 |                      |
                 |                      +-'=='
                 |                      |
                 |                      +-'y'
                 |
                 +-then-stmt---assign-+-'z'
                 |                    |
                 |                    +-'1'
                 |
                 +-else-stmt---assign-+-'z'
                                      |
                                      +-'2'
    ```

* Semantic Analysis: understand the "meaning" of sentences.

    It's too hard for compilers and compilers usually only perform limited semantic analysis to catch inconsistencies, they don't know what programs are supposed to do.

    > **The Complexity of Semantics**
    >
    > "Jack said that Jack forgot his homework at home". We even don't know how many people are involved in this sentence without contexts.
    >
    > Programming languages define strict rules to avoid such ambiguities. e.g.
    >
    > ```c
    > {
    >  int jack = 3;
    >  {
    >      int jack = 4;
    >      cout << jack; // should print 4
    >  }
    > }
    > ```

* Optimization: automatically modify programs so that they run faster and use less memory, power, network resources etc.

    > **The complexity of Optimizations**
    >
    > `X = Y * 0` is the same as `X = 0` ? It's not necessarily correct! If `X` `Y` are floating points and `Y` is NAN, then NAN * 0 = NAN, instead of zero.

* Code Generation: a translation into another language (assembly usually).

The proportions of each phase have changed since FORTRAN:

* FORTRAN: `LLLLLL PPPPPP SS OOOOOO GGGGGG`
* Modern: `LL PP SSSSSS OOOOOOOOOO GG`

## The Economy of Programming Languages

### Why so many PLs? 

Application domains have distinctive/conflicting needs. It's hard to design one system for all.

* scientific computing: good floating points, good array/tensor operations, parallelism etc. (FORTRAN)
* business applications: persistence, report generation, data analysis etc. (SQL)
* systems programming: control of low-level resources, real-time constraints (C/C++)

### Why new PLs? 

Claim: Programmer training is the dominant cost for a programming language. (Lots of people need to learn this language!)

Corollaries:

* Widely-used languages are slow to change.
* Easy to start a new language (zero user at the start).
* New languages are usually adopted to fill a void (it's easy for new language to adapt to new situations).
* New languages tend to look like old languages (to reduce the training cost).

### What's a good PL?

Thers is no universally accepted metric for language design.