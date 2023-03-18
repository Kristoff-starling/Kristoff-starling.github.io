---
title: "Stanford-CS143 Lecture 03: Lexical Analysis"
linktitle: 'Lecture 03'
date: 2022-07-10
type: docs
draft: false

math: true

menu:
    stanford-compiler-lectures:
        parent: Contents
        weight: 3
---

Token classes:

* In English: nouns, verbs, adjactives etc.
* In programming languages: identifiers, keywords, "()" etc.
    * Identifier: strings of letters or digits, staring with a letter (variable names).
    * Integer: a non-empty string of digits (leading zeroes are allowed).
    * Keywords: "if", "else", "begin" etc.
    * Whitespace: a non-empty sequence of blanks, newlines and tabs
    * Operator: relational operators like `==` `<` `>` etc.
    * `(` `)` `;` `=` : these are special token classes which consist of only one string.

A key-value pair of format `<class, string>` is called a token. The lexical analyzer takes in a string(program) and outputs tokens to the parser. For example：

* Input: `foo=42`
* Output: `<identifier, "foo">` , `<operator, "=">` , `<integer, "42">` (note: here "42" is a string, not a number!)

An implementation of LA must do two things:

* Recognize substrings corresponding to tokens, which are called lexemes.
* Idenfity the token class of each lexeme.

## LA Examples

FORTRAN rule: whitespace is insignificant. So `VAR1` is the same as `VA  R  1`.

An Example:

```fortran
DO 5 I = 1, 25
DO 5 I = 1.25
```

The first line is a do-loop where variable `I` iterates from 1 to 25. The second line, however, is a variable definition - `DO5I = 1.25`. When we scan the string from left to right, at tht point we finish "DO" we're not sure whether it's a keyword, we must look ahead to see whether there's a comma. Therefore, **"lookahead" may be required to decide where one token ends and the next token begins**.

"lookahead" is a universal need even in modern PLs. For example,

```c
if (i == j) x = 1; else x = 2;
```

when we encounter `=`, we need to look ahead to decide whether it's a double-equal; when we encounter `e` we need to look ahead to decide whether it's a variable name or a keyword.

PL/1 rule: keywords are not reserved (i.e. keywords can be used as variable names).

```
IF ELSE THEN THEN = ELSE; ELSE ELSE = THEN
```

It's challenging to recognize which `ELSE`&`THEN` are variables.

```
DECLARE(ARG1, ..., ARGN)
```

we're not sure whether `DECLARE` is a keyword or an array reference. Since it can have arbitrary number of arguments, an <u>unbounded lookahead</u> will be required.

## Regular Languages

We must specify what set of strings is in a token class, and the solution is to use regular languages.

2 base cases:

* Single character: $c\triangleq \{c\}$
* Epsilon: $\epsilon\triangleq \{''\}$

3 compound expressions:

* Union: $A+B\triangleq \\{a|a\in A\\}\cup \\{|b\in B\\}$
* Concatenation: $AB\triangleq \\{ab|a\in A,b\in B\\}$
* Iteration: $A^* \triangleq \left\\{\bigcup_{i\geq 0}A^i \right\\}$, here $A^0=\epsilon$.

$\fbox{Definition 3.1}$ The **regular expressions** over $\Sigma$ are the smallest set of expressions including:
$$
\begin{align}
R::=&\quad\space \epsilon\\\\
&|\quad c, c\in \Sigma\\\\
&|\quad R + R\\\\
&|\quad RR\\\\
&|\quad R^*
\end{align}
$$

## Formal Languages

$\fbox{Definition 3.2}$ Let $\Sigma$ be a set of characters (an **alphabet**), a **language** over $\Sigma$ is a set of strings of characters drawn from $\Sigma$.

e.g. Alphabet = English characters, Language = sentences (maybe we need a strict definition of valid sentences)

e.g. Alphabet = ASCII, Language = C programs (this is rigorous, the language is exactly what C compilers accept)

A **meaning function** $L:\text{expression}\to \text{set of strings}$ is a mapping that maps syntax to semantics. For example, the rigorous way of defining regular expression symbols are:
$$
\begin{align}
L(\epsilon)&=\{''\}\\\\
L(c)&=\{c\}\\\\
L(A+B)&=L(A)\cup L(B)\\\\
L(AB)&=\{ab:a\in L(A),b\in L(B)\}\\\\
L(A^*)&=\bigcup_{i\geq 0}L(A^i)
\end{align}
$$
The mappings between expressions and meanings are not 1-to-1. it's many-to-1: e.g. $L(11+10)=L(1(1+0))$. This is the foundation of optimization: we can use simpler expressions to do exactly the same thing. But meaning functions should never be 1-to-many, indicating that the language is ambiguous.

Meaning function makes clear what is syntax and what is semantics. It allows us to consider notation as a separate issue. It's important to think about syntax/notation separately. For example, Roman numerals and Arabic numerals are two systems having the same semantics, but the latter is popular because of its simplicity in syntax.

## Lexical Specifications

### Lexemes

We use regular expressions to specify lexemes.

* Keywords: if/else/then/...

    $\text{Keywords}=\text{if}+\text{else}+\text{then}+\cdots$

* Integer: a non-empty string of digits.

    $\text{digit}=0+1+2+3+\ldots+8+9$，$\text{Integer}=\text{digit}\space \text{digit}^*$.

    Representing non-empty strings is a common need, so we denote $A^+\triangleq AA^*$. Therefore we can also write $\text{Integer}=\text{digit}^+$.

* Identifier: strings of letters of digits, starting with a letter.

    $\text{letter}=a+b+\cdots+y+z+A+B+\cdots+Y+Z$. There is a convenient notation for ranges: $\text{letter}=[a- z]+[A- Z]=[a- zA- Z]$.

    $\text{Identifier}=\text{letter}\space (\text{letter}+\text{digit})^*$.

* Whitespace: a non-empty sequence of blanks, newlines and tabs.

    $\text{Whitespace}=(\text{' '}+\text{'\\n'}+\text{'\\t'})^+$.

### Example: Pascal Numbers

$$
\begin{align}
\text{num} &= \text{digits}\space\space \text{opt\_fraction}\space\space \text{opt\_exponent}\\\\
\text{opt\_fraction}&=(\text{'.'}\space\space \text{digits})+\epsilon\\\\
\text{opt\_exponent}&=(\text{'E'}\space (\text{'+'}+\text{'-'}+\epsilon)\space \text{digits})+\epsilon\\\\
\text{digits}&=\text{digit}^+\\\\
\text{digit}&=[0-9]
\end{align}
$$

We observe that "$+\epsilon$" in regular expressions indicates "optional", i.e. the elements before can be either present or absent. For convenience, we denote $A?\triangleq A+\epsilon$, so the rules above can be rewritten as
$$
\begin{align}
\text{opt\_fraction} &= (\text{'.'}\space \text{digits})?\\\\
\text{opt\_exponent} &= (\text{'E'}\space (\text{'+'}+\text{'-'})?\space \text{digits})?
\end{align}
$$
It's worth notice that regular expressions defines the set of strings we want, but we still need an implementation, i.e. given a string $s$ and a regular expression $R$, we want to know whether $s\in L(R)$.

