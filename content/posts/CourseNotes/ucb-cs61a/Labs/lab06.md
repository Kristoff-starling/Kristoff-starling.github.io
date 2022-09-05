---
title: "UCB-CS61A Lab 06: Mutability and Iterators"
linktitle: 'Lab 06'
type: docs
draft: false

math: true

menu:
    cs61a-lab:
        parent: Contents
        weight: 6
---

## List API

一些重要的 API 的补充说明：

* `list.pop(index=-1)` 的作用是弹出 `list[index]` 这个元素，没有填写参数默认为 -1。该 method 的返回值是弹出的元素的值。
* `list.remove(x)` 删除 list 中的第一个元素 x。
* `list.insert(index, obj)` 的作用是在 index 为 `index` 的元素前面添加元素 obj (即插入完元素后 obj 的下标是 index)
* `list.append()` 的返回值是 None。 一道例题很好地展现了这个特性：

    ```python
    >>> lst = [1, 2, 3]
    >>> lst.extend(lst.append(4), list.append(5))
    >>> lst
    [1, 2, 3, None, None]
    ```

## Iterators

两个值得注意的细节：

* 如果一个 iterater `iter` 在 for 循环中被使用，例如 `for x in iter`，那么 for 循环结束时 iter 的值是被改变了的，再调用 `next()` 会得到 StopIteration。
* 使用 `list(iter)` 获得列表的话会修改 iter 的位置，所以如果只想取前 n 个元素且保证 iter 的位置，不能使用 `list(iter)[:n]` 的方式。

