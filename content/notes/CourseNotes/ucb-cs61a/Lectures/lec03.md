---
title: "UCB-CS61A Lecture 03: Control"
linktitle: '03: Control'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 3
---

## Side Effects

一个函数如果没有显式地返回内容，那么它的返回值默认为 None。比如下面的程序会报 TypeError：

```python
def square(x):
    x * x

result = square(4) # result = 'None'
print(result + 1)
```

一个函数有 side-effect 指的是它除了返回内容还做了一些可见的事情，最常见的有 side-effect 的函数就是 `print()`。

> **一个有趣的问题：`print(print(1), print(2))` 的结果？**
>
> 首先，里面的 `print(1)` 会 Display "1"，`print(2)` 会 Display "2"。两个 print 函数没有显式地返回，所以外层的 print 会 Display "None None"。整个语句的返回值是 None。

