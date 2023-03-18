---
title: "UCB-CS61A Lecture 17: Generators"
linktitle: '17: 生成器'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 17
---

## Generators

generator function 中不使用 return 返回结果，而使用 yield。generator 是一种 iterator，它通过 generator function 来逐个地获得结果。这里 generator function 可以理解为一个状态机，当遇到 yield 语句时，它会将自己的执行状态暂时封存起来并返回结果。下一次 generator 再次调用该函数时，generator function 将会从上一次 yield 后的第一条语句开始继续执行，直到遇到下一个 yield。如果没有遇到 yield 而是函数一直执行到结束，generator function 会返回一个 `StopIteration` exception。

下面是一个例子：

```python
def evens(start, end):
    num = start + (start % 2)
    while num < end:
        yield num
        num += 2

for num in evens(55, 61):
    print(num)
# 56
# 58
# 60
```

这个例子的功能和

```python
evens = [num for num in range(55, 61) if num % 2 == 0]
```

相同。使用 generator 的好处时 generator function 每次只生成下一个元素。如果列表很长，直接生成出整个列表再遍历的方式会消耗很多内存。

## Yield From

`yield from` 语句可以将一个 iterable 中的元素逐个 yield 出去。例如下面两段程序的功能等价：

```python
def yield_list(a):
    for item in a:
        yield item
```

```python
def yield_list(a):
    yield from a
```

更有趣的用法是：`yield from` 后面跟的东西可以是 generator function，甚至是自己本身这个 generator function，我们可以利用这个性质写出一些类似于”递归“ 的 generator function。下面是一个”递归“版本的 countdown：

```python
def countdown(k):
    if k > 0:
        yield k
        yield from countdown(k - 1)
```

## Generator Functions with Returns

如果 generator function 执行过程中遇到 return，那么即使下次再调用后面的内容也不会再执行，return value 不会作为 yield 的一部分。如果你确实想要将返回值也 yield 出来，可以这样写：

```python
def g(x):
    yield x
    yield x + 1
    return x + 2
def h(x):
    r = yield from g(x)
    yield r
list(h(2)) # [2, 3, 4]
```

## Example

下面展示一个用 generator function 来生成所有整数 partition 的例子：

```python
def partitions(n, m):
    """List partitions.

    >>> for p in partitions(6, 4): print(p)
    4 + 2
    4 + 1 + 1
    3 + 3
    3 + 2 + 1
    3 + 1 + 1 + 1
    2 + 2 + 2
    2 + 2 + 1 + 1
    2 + 1 + 1 + 1 + 1
    1 + 1 + 1 + 1 + 1 + 1
    """
    if n < 0 or m == 0:
        return
    else:
        if n == m:
            yield str(m)
        for p in partitions(n-m, m):
            yield str(m) + " + " + p
        yield from partitions(n, m - 1)
```

