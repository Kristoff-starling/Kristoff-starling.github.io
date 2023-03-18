---
title: "UCB-CS61A Lecture 16: Iterators"
linktitle: '16: 迭代器'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 16
---

## Iterators

迭代器 (iterator) 是一种对象，提供了对某个 itarable 的 sequential access。`iter(iterable)` 会返回一个指向 iterable 第一个元素的 iterator，之后可以用 `next()` 函数来查看后续的元素。

```python
toppings = ["pineapple", "pepper", "mushroom", "roasted red pepper"]

topperator = iter(toppings)
next(topperator) # 'pineapple'
next(topperator) # 'pepper'
next(topperator) # 'mushroom'
next(topperator) # 'roasted red pepper'
next(topperator) # ❌ StopIteration exception
```

注：如果给 `iter()` 传入的参数是 iterator，那么它会返回一个一样的 iterator。

```python
>>> topperator2 = iter(topperator)
>>> topperator is topperator2
True
```

list, tuple, dictionary, string, range 都属于 iterable，这里提一下对于字典的访问：从 Python 3.6 开始，字典中的元素按照它们插入的顺序排列，使用 `iter(dic.keys())` 可以获得 key 的 iterator；使用 `iter(dic.values())` 可以获得 value 的 iterator；使用 `iter(dic.items))` 可以获得 key-value pair 的 iterator。

如果在 for 循环中使用 iterator，那么每次执行完循环体后 Python 都会调用 `next()` 让 iterator 指向下一个元素，直到没有更多元素：

```python
nums = range(1, 4)
num_iter = iter(nums)
first = iter(num_iter) # 1
for num in num_iter:
    print(num)
# 2
# 3
# 4
```

## Useful Bulit-in Functions

一些比较好用的返回 iterable 的函数：

* `list(iterable)`：返回一个 list，该 list 中的元素是 iterable 中的元素 (这里传入的也可以是一个 iterator)。
* `tuple(iterable)`：类似，返回一个 tuple。
* `sorted(iterable)` ：返回一个 list，里面的元素是排好序的。

一些比较好用的返回 iterator 的函数：

* `map(func, iterable)` 会对 iterable 中的每个元素 x 执行 `func(x)`，并返回一个 iterator。我们可以用 `list(map(func, iterable))` 的方式来获得列表，这句话的功能和 `[func(x) for x in iterable]` 等价。

* `filter(func, iterable)` 会保留 iterable 中那些满足 `func(x)` 为 True 的元素 x，并返回一个 iterator。`list(filter(func, iterable))` 的功能和 `[x for x in iterable if func(x)]` 等价。

* `zip(*iterables)` 接收两个 iterable，将这个 iterable 处于相同下标的元素组成 pair，返回一个 pair list 的 iterator。例如：

    ```python
    num1, num2 = [1, 2, 3], [4, 5, 6]
    num_pairs = list(zip(num1, num2))
    for n1, n2 in num_pairs:
        print(n1, n2)
    # 1 4
    # 2 5
    # 3 6
    ```