---
title: "Stanford-CS143 PA0: Cool Programming"
linktitle: 'PA0: Cool'
type: docs
draft: false

math: true

menu:
    stanford-compiler-pa:
        parent: Contents
        weight: 1
---

该入门实验的主要目的是熟悉 cool 语言的语法，为编译器编写打下基础。本实验的任务是用 cool 语言写一个简单的 stack machine，支持 `int` `+` `s` `e` `d` 五种操作。

## Designation

由于 cool 语言中没有开数组的功能，所以笔者使用链表来模拟这个栈数据结构。插入操作是新建一个链表节点插在 head 前面，`e` 操作则是查看链表开头的若干个元素并执行相应操作。

## Interesting Bugs

cool 是一个面向对象风格的语言，一开始笔者吃了不少苦头；而且 PA 提供的 cool 编译器报错信息很少，不方便知道到底发生了什么，只能自己猜测和摸索。

笔者遇到的一个有意思的 bug 是在实现 `class List` 的某个 method 时，无法直接使用另一个 List 变量的 attribute。这是因为 cool 要求所有的 attribute 都是 private 的，method 是 public 的，因此如果想要访问某个变量的 attribute 要这样写：

```java
class CLASS {
    attribute : Object;
    
    attribute() : Object { attribute };
};
```

这样就可以通过 attribute() 方法来访问 attribute 属性。

## Miscellaneous

关于 cool 的一点吐槽 (README 要求的作业之一)：

* 无法指定 private/public 的 attribute；

* 没有逻辑运算符 (and/or) 导致有一些 if-else 语句不得不拆成两个写；
* if 结构必须是 if-then-else-fi，不能省略 else 分支；
* 多行代码必须用大括号框起来，有一点丑 (不过这应该可以简化 parser 的设计)；
* ……