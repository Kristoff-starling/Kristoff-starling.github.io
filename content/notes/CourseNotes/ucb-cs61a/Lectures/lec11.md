---
title: "UCB-CS61A Lecture 11: Sequences"
linktitle: '11: Sequences'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 11
---

## Copying

我们平时写的

```python
listA = [0, 1, 2, 3]
```

本质上是一个名叫 "listA"  的指针指向了 `[0, 1, 2, 3]` 这个对象。

拷贝分深拷贝和浅拷贝两种。浅拷贝只是让两个指针指向了同一个对象，而深拷贝会将对象复制一遍，然后让新指针指向新复制的对象。例如：

```python
>>> listA = [0, 1, 2, 3]
>>> listB = listA         # 浅拷贝
>>> listC = list(listA)   # 深拷贝
>>> listA[0] = 1
>>> listB, listC
([1, 1, 2, 3], [0, 1, 2, 3])
```

## Built-in Functions for Iterables

一些常见的内置函数有：

| Function                  | Description                                     |
| ------------------------- | ----------------------------------------------- |
| `sum(iterable)`           | 求和                                            |
| `all(iterable)`           | 返回 True 当且仅当 iterable 中所有元素均为 True |
| `any(iterable)`           | 返回 True 当且仅当 iterable 中存在元素为 True   |
| `max(iterable, key=None)` | 最大值                                          |
| `min(iterable, key=None)` | 最小值                                          |

其中 `max()` `min()` 可以通过给 key 传递函数的方式自定义比大小的标准，例如：

```python
>>> coords = [ [37, -144], [-22, -115], [56, -163] ]
>>> max(coords, key=lambda coord: coord[0])
[56, -163]
```