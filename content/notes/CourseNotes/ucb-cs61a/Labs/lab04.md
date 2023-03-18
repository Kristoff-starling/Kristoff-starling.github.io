---
title: "UCB-CS61A Lab 04: Recursion, Tree Recursion, Python Lists"
linktitle: 'Lab 04'
type: docs
draft: false

math: true

menu:
    cs61a-lab:
        parent: Contents
        weight: 4
---

下面的两道题是比较有意思的：

## Q1: WWPD: Journey to the Center of the Earch

```python
>>> def crust():
...     print("70km")
...     def mantle():
...          print("2900km")
...          def core():
...               print("5300km")
...               return mantle()
...          return core
...     return mantle
>>> drill = crust
>>> drill = drill()
______

>>> drill = drill()
______

>>> drill = drill()
______

>>> drill()
______
```

值得注意的是第三次和第四次调用 `drill()` 的区别。两次调用的 `drill()` 都是 `core()` 这个函数，它们首先会输出 "5300km"，然后调用 `mantle()` 之后输出 "2900km"，最后返回函数 `core()`。但第一次的语句是一个赋值语句，赋值语句的返回值是 None，第二次的语句是一个单纯的调用，所以最后函数的返回值 "Function..." 也会被打印出来。

## Q7: Riffle Shuffle

这道题有一点小难度，因为 list comprehension 中不能带有 else，所以我们要想出一种统一的方法来根据下标从数组中取数。我们可以利用 `i % 2` 的结果参与运算：

```python
return [deck[i // 2 + (i % 2) * len(deck) // 2] for i in range(len(deck))]
```

