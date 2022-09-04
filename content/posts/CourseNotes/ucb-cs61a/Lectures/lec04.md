---
title: "UCB-CS61A Lecture 04: Higher-Order Functions"
linktitle: '04: 高阶函数'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 4
---

## Higher-Order Functions

以函数作为参数的，或者以函数作为返回值的函数被称为 higher-order function。例如

以函数作为参数：

```python
def cube(k):
    return k ** 3
def summation(n, term):
    """Sum the first N terms of a sequence
    >>> summation(5, cube)
    225
    """
    total, k = 0, 1;
    while k <= n:
        total = total + term(k)
        k += 1
    return total
```

传入的 `term` 是一个函数。

以函数为返回值：

```python
def make_adder(n):
    """Return a function that takes one argument k
       and returns k + n
    >>> add_three = make_adder(3)
    >>> add_three(4)
    7
    """
    def adder(k):
        return k + n
   	return adder
```

这样我们就可以将一个 call expression 作为一个 operator。例如

```python
make_adder(1)(2)
```

其中 `make_adder(1)` 形式上是 call expression，但返回结果是 `adder` 这一 operator，因此最终会返回 3。

## Lambda Expression

lambda 表达式定义了一个匿名函数，其格式为

```python
lambda <parameters>: <expression>
```

lambda 表达式作为函数参数传给高阶函数是很方便的，例如之前的 summation 的例子中我们可以写

```python
summation(5, lambda x: x ** 3)
```

从而不需要额外定义 `cube` 这个函数。

带有条件的 lambda 表达式举例 (取绝对值函数)：

```python
lambda x: x if x > 0 else -x
```

