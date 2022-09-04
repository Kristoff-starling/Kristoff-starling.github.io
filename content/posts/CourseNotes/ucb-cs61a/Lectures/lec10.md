---
title: "UCB-CS61A Lecture 10: Containers"
linktitle: '10: 容器'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 10
---

## List Comprehension

通过一个已经存在的 list 创建一个新的 list。

完整的语法为：

```python
[<map exp> for <name> in <iter exp> if <filter exp>]
```

在执行这样一条语句时，Python 解释器会做这样的事情：

* 临时创建一个新的 frame，这个 frame 的 parent frame 是执行该语句的当前 frame。
* 创建一个空的 list 作为当前语句的结果
* 执行 `<iter exp>`，结果必须是一个 iterable 的东西。
* 对于 `<iter exp>` 结果中的每个元素，将 `<name>` 绑定到该元素上，并执行 `<filter exp>` 以决定是否放入 list。
* 临时 frame 的返回值就是结果 list。

注意：list comprehension 中的 if 语句不能跟 else。