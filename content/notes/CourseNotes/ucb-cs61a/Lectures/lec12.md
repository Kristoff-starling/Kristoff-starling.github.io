---
title: "UCB-CS61A Lecture 12: Data Abstraction"
linktitle: '12: 数据抽象'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 12
---

## Abstraction Barriers

抽象是一层一层的，好的程序设计应当只使用上一层抽象提供的接口，不应该跨层地去使用底层的接口。

## Dictionary

### Dictionary Selection

一个有用的 API：

```python
>>> people = {'alice': 'girl', 'bob': 'boy'}
>>> people.get("eve", "🤔")
🤔
```

`dict.get(key, default=None)` 会返回字典 `dict` 中 key 对应的 value，如果 key 不存在则会返回 default 的内容。

### Dictionary Rules

字典的 key 不能是 list, dictionary 等 mutable 的东西，且 key 是两两不同的，但 value 可以是任意内容。

### Dictionary Iteration

```python
>>> insects = {"spiders": 8, "centipedes": 100, "bees": 6}
>>> for name in insects:
>>>     print(insects[name])
8 100 6
```

遍历字典时按照 key-value pair 被加入字典时的顺序访问。

### Dictionary Comprehensions

General syntax:

```python
{key: value for <name> in <iter exp>}
```

Example:

```python
squares = {x: x * x for x in range(3, 6)}
```

