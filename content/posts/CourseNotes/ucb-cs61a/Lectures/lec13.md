---
title: "UCB-CS61A Lecture 13: Trees"
linktitle: '13: 树'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 13
---

## Tree: Data Abstraction

我们希望 tree 这个抽象数据类型提供如下的几个 API:

* `tree(label, branches)`：创建以 `label` 为根，`branches` 为子树的树 (`branches` 是一个列表，其中的每个元素都是一个 tree)。
* `label(tree)`：返回一个树 tree 的根。
* `branches(tree)` 返回一个树的子树的列表。
* `is_leaf(tree)` 判断 tree 是不是一个叶子 (没有子树)。

## Tree: A Simple Implementation

使用嵌套的列表去实现：

```python
def tree(label, branches=[]):
    return [label] + list(branches)
def label(tree):
    return tree[0]
def branches(tree):
    return tree[1:]
def is_leaf(tree):
    return len(branches(tree)) == 0
```

## Exercise: List of Leaves

```python
def leaves(t):
    """Return a list containing the leaf labels of T.
    >>> t = tree(20, [tree(12, [tree(9, [tree(7), tree(2)]), tree(3)]), tree(8, [tree(4), tree(4)])])
    >>> leaves(t)
    [7, 2, 3, 4, 4]
    """
    if is_leaf(t):
        return [label(t)]
    else
    	leaf_labels = [leaves(b) for b in branches(t)]
        return sum(leaf_labels, [])
```

注：`sum()` 函数如果接受两个 list，其中第一个 list 里是若干个 list，那么它的行为是将第一个 list 中的所有 list 中的元素整合成一个大 list，接在第二个参数的 list 后面。