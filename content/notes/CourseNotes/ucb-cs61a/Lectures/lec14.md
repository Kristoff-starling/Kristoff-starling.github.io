---
title: "UCB-CS61A Lecture 14: Mutability"
linktitle: '14: Mutability'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 14
---

## Objects

对象 (object) 是一组数据和行为 (函数) 的集合。在 Python 中所有的 value 都是对象，对象有一系列属性 (attribute) 也有一系列方法 (method)。比如 Python 中的字符串可以通过索引的方式获取具体位置的数据，也可以使用 upper(), lower() 等方法对其进行操作。

## List Mutation

`append()` 和 `extend()` 这两个方法的行为是有区别的：

* `append()` 只能向 list 的后面添加一个元素。如果 `append()` 的参数是一个 list，那么它会在本 list 的后面添加一个元素，这个元素是一个 list (形成了一个嵌套的 list)
* `extend()` 的参数必须是一个 iterable 的东西，它会将这个东西里的所有元素添加到本 list 中。

```python
>>> s1, s2 = [2, 3]
>>> t = [4, 5]
>>> s1.append(t)
>>> s2.extend(t)
>>> s1
[2, 3, [4, 5]]
>>> s2
[2, 3, 4, 5]
```

`remove(v)` 方法用于删除 list 中的第一个元素 v (如果 list 中没有 v 会报错)。

## Tuples

tuple 和 list 基本相同，除了 tuple 中的元素不可修改。创建 tuple 应使用小括号。值得注意的是如果要创建只有一个元素 v 的 tuple，应当写成 `(v,)` 而不是 `(v)`。

list 中一些只读的操作 tuple 中同样有，比如 slicing，`+` 合并，`in` 判断某个元素是否存在等等。

## Immutability vs. Mutability

immutable 的 value 一旦创建就不能修改。常见的 immutable 的例子有 int, float, string, tuple 等。注：我们平时可以写 `a += 2` 这种操作修改变量的值，本质上是让 a 这个变量名与一个新的 int value 捆绑，而不是对原值的修改。

如果一个 imuutable 的对象里的元素是 mutable 的 (比如 tuple 的元素是 list)，那么该元素是可以修改的：

```python
>>> t = (1, [2, 3])
>>> t[1][0] = 99
>>> t[1][1] = "problem"
>>> t
(1, [99, 'problem'])
```

**Equality of contents vs. Identity of objects**

`==` 和 `is` 这两个比较方法是有区别的。`==` 的返回值为 True 当且仅当两个对象的 value 相同。而 `is` 要求两个变量名指向同一个对象。`is` 返回值为 True 的 `==` 必定为 True，反之则不一定。

```python
>>> list1, list2 = [1, 2, 3], [1, 2, 3]
>>> list1 == list2
True
>>> list1 is list2
False
>>> list2 = list1
>>> list1 is list2
True
```

## Mutable Functions

Python 函数可以对 parent frame 中的 mutable 的对象进行修改。下面是一个例子：

```python
def make_withdraw_account(initial):
    balance = [initial]
    
    def withdraw(amount):
        if balance[0] - amount < 0:
            return 'Insufficient funds'
        balance[0] -= amount
        return balance[0]
    
    return withdraw
```

注：如果我们想要用一个变量 `balance` 来存储金额，那么在 `withdraw()` 函数中就要用 `nonlocal balance` 来声明下面的 balance 是对 parent frame 中的变量进行的操作。