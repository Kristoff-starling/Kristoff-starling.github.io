---
title: "UCB-CS61A Lecture 07: Function Examples"
linktitle: '07: 函数举例'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 7
---

## Decorators

考虑如下的高阶函数 `trace()` ：

```python
def square(x):
    return x * x
def trace(f):
    def traced(x):
        print(f'arg: {x}')
        r = f(x)
        print(f'return: {r}')
        return r
    return traced
```

当我们调用 `trace(square)` 时，我们就会获得一个函数，这个函数的功能和 `square()` 是一样的，但传入的参数和返回值会被打印出来。

如果我们想要每次调用 `square()` 时都使用这个 traced 的版本，每次都写 `trace(square)` 有点太麻烦了。这时我们可以使用**装饰器 (decorator)**：

```python
@trace
def square(x):
    return x * x
```

这样调用 `square()` 时就会自动 trace 参数和返回值。装饰器的原理是：如下的代码

```python
@ATTR
def func(...):
    ...
```

等价于

```python
def func(...):
    ...
func = ATTR(func)
```

