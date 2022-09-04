---
title: "UCB-CS61A Lecture 05: Environments"
linktitle: '05: 环境'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 5
---

## Environments

* 刚开始有一个 global frame，每发生一次函数调用，就会随之生成一个 local frame。
* 每个用户定义的 function 都有一个 parent frame。一个函数的 parent 指的是函数定义所在的 frame。
* 每个 frame 也有一个 parent frame，一个 frame 的 parent 指的是生成这个 frame 的函数的 parent。
* environment 指的是一串 frame 序列。

### Environment Diagram

当一个函数被定义时：

* 创建一个 function value：`func <name>(<formal parameters>) [parent=<label>]`，这里的 parent 就是当前的 frame。
* 当前 frame 中的 `<name>` 指向创建的 function value。

当一个函数被调用时：

* 添加一个 local frame，frame 的名字是被调用的函数的名字，frame 的 parent 是被调用的函数的 parent。
* `<formal parameters>`  指向传入的参数。

## Self-Reference

高阶函数可以拿自己作为返回值，例如

```python
def print_sums(n):
    print(n)
    def next_sum(k):
        return print_sums(n + k)
    return next_sum
```

调用 `print_sums(1)(3)(5)` 后会打印出 `1\n 4\n 9\n`，且最后的返回值仍然是一个 `next_sum()` 函数。

## Function Currying

Currying 指的是将接收多个参数的函数转化为只接收一个参数的高阶函数。例如我们可以实现一个这样的函数：

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
       	return h
   	return g
```

或者用 lambda 表达式：

```python
curry = lambda f: lambda x: lambda y: f(x, y)
```

调用 `curry(add)(2, 3)` 和调用 `add(2, 3)` 的效果一样。利用 Currying 创建一些中间状态的辅助函数可以减少重复传参。