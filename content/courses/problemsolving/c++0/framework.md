---
title: "基本框架"
linktitle: "基本框架"
type: docs
draft: false

math: true

menu:
    c++0:
        parent: Contents
        weight: 1
---

一个最简单的 C++ 程序长成这样：
```c++
#include <bits/stdc++.h>

int main ()
{
    // your code
    return 0;
}
```
我们对这个程序的几个组成部分做一点说明：
* "include"一行使程序包含了一系列**头文件 (header file)**, 头文件中定义了许多有用的函数，我们只有使用 include 包含这些头文件才能使用这些函数（头文件在安装环境时就有了，你暂时不需要关心它们在哪里以及是如何实现的，你只需要知道 include 这行几乎是必须要写的）。
* `int main () {}` 称为 main 函数。每个程序都必须有 main 函数，当程序开始运行时，第一条执行的指令就是 main 函数的第一条指令。
* `return 0;` 是一条语句，无论 main 函数中写了什么内容，最后一样都应当是 `return 0;`。

> **C++ 语法** 
>
> * 你的每条语句都**必须**以 `;` 结尾。一行可以有多条语句，但每个语句后都要有 `;`。
> * 使用 `//` 可以在 C++ 代码中书写**注释 (comment)**，注释类似于批注，其目的是让阅读代码的人更好地理解代码的意思，注释中的内容不会被执行。此外，如果你想书写一段多行的注释，可以使用如下方法：
>
>    ```c++
>    /*
>      write your comments here
>      write your comments here
>    */
>    ```

