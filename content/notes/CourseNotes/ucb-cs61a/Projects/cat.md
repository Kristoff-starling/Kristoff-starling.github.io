---
title: "UCB-CS61A Project 02: Cat"
linktitle: '02: Cat'
type: docs
draft: false

math: true

menu:
    cs61a-projects:
        parent: Contents
        weight: 2
---

## Phase 1: Typing

### Problem 1

寻找第 k 个满足条件的字符串，注意这里的 k 是 0-base 的。

### Problem 2

充分利用 `utils.py` 中提供的 `split()` `remove_punctuation()` 和 `lower()` 函数可以大幅简化代码的难度。

### Problem 3

* 注意单独处理几种列表长度为 0 的边界情况。
* 返回值 0/100 会被当作一个整数，想要返回一个浮点数应当写 0.0/100.0。

### Problem 4

简单数学计算。

## Phase 2: Autocorrect

### Problem 5

本题比较漂亮的写法是充分运用 `min()` 函数的 key：直接把 `diff_function()` 的内容用 lambda 表达式套起来喂给 key，可以省去书写 for 循环。

### Problem 6

本题的难点在于不能使用循环只能用递归，此外还要保证答案超过 limit 的时候运行效率尽可能与长度无关。笔者实现的方法是将答案作为参数随着递归传下去，这样一旦答案超出 limit 递归就提前返回。

### Problem 7

这题颇具难度，其思路类似于动态规划。一个比较简单的做法是：我们每次考虑如何让两个单词的第一个字母匹配上。如果已经一样就跳过看下一个字母，否则就从加字母 (肯定会加匹配的字母)，删字母 (删字母后并不保证第一个字母能匹配上)，换字母中选一种操作。

## Phase 3: Multiplayer

### Problem 8

本题可以关注字典的初始化方法：在 `{}` 中用 `'key': value` 的方式包括若干 key-value pairs。

### Problem 9

本题没有太大的难度，运用 list comprehension 可以将代码写得简短漂亮。

### Problem 10

该题本身没有太大难度，但应当注意实现时最好使用框架代码提供的几个和 match 互动的 API，将 match 当作一个 abstract data structure 而不是直接去定位嵌套列表中的元素。所谓的 abstract data structure 就是只关注 API 提供的数据结构对外的交互功能而忽略数据结构内部的实现方法。

## Utils

`utils.py` 提供了一些辅助函数。其中值得注意的 API 用法有：

```python
def remove_punctuation(s):
    punctuation_remover = str.maketrans('', '', string.punctuation)
    return s.strip().translate(punctuation_remover)
```

`str.maketrans()` 为 `translate()` 提供字符转换的规则：

* 如果 `maketrans()` 接收一个参数，那么这个参数必须是一个字典。字典的映射就是转换的源和目标。
* 如果 `maketrans()` 接收两个参数，那么这两个参数必须是长度相同的字符串，对应的字符构成转换规则。
* 如果 `maketrans()` 接收三个参数，那么第三个参数的字符串的内容会被映射到 None，前两个同 (2)。

这里主要利用了三个参数的规则将所有的标点符号映射到了 None。

`strip(c=' ')` 方法用于去除字符串首尾的字符 c，如果没有传入参数则默认去除空格。

