---
layout: post
date: 2022-03-11
title: "NJU-OS MiniLab 1: Process Tree"
categories: ["NJU-22020230 Operating Systems", "Lab Reports"]
tags: 
mathjax: true
---

该实验主要目的是实现一个精简版的 Linux pstree 命令行工具，以较为优美的方式打印当前系统中的进程树。

<!-- more -->

## Parse args

首先对输入的选项列表进行分析。笔者使用了 C 库函数 getopt_long()，它不仅可以帮助我们快速解析选项，还可以同时识别一个选项的长短版本 (使用 C 库提供的 option 结构体可以定义每种选项的长短版本以及是否需要参数等)，识别多个选吸纳更合并给出的格式 etc.。

定义好表格后，仅需一句话即可完成选项解析：

```c
const struct option table[] = {
    {"numeric-sort", no_argument, NULL, 'n'},
    {"show-pids"   , no_argument, NULL, 'p'},
    {"version"     , no_argument, NULL, 'V'},
    {0             , 0          , NULL,  0 },
};

getopt_long(argc, argv, "-npV", table, NULL)
```

> **关于版本信息**
>
> Linux 的 pstree 工具会将版本信息输出到标准错误流，为了模拟这一特性，我们可以使用 fprintf 语句将版本信息输出到 stderr 文件。

## Fetch Process Information

Linux 提供 procfs，我们可以用读写文件的 API 来获取操作系统中的进程状态。procfs 的根目录是 `/proc`，该目录中所有是数字的文件夹都存储了一个进程相关的信息，文件夹的数字名就是进程编号。配合使用 C 库函数 opendir() 和 readdir() 就可以遍历 `/proc` 下的所有内容并筛选出那些是数字的文件夹名。

每个进程文件夹下的 `/proc/[pid]/stat` 文件中存储了我们打印进程树所需要的状态信息，包括但不限于进程的名字，进程的状态，进程号，父进程号等。我们可以利用这些信息把进程树构建出来。

> **关于进程名的格式**
>
> 根据 `man proc` 的提示，所有的进程名都是用一对小括号包裹着的。笔者起初使用了 `scanf("%[^)]")` 的方法，读到右括号停止，但发现进程中有一些进程的进程名本身就带括号。因此只能一个字节一个字节读并同步维护一个括号的栈来确定什么时候停止。

我们可以在 `/proc/sys/kernel/pid_max` 文件中查看进程号的最大值，在 64 位系统中，这个最大值为 $2^{22}$。为了代码编写的方便和效率，笔者开了一个这么大的数组 proc_pos[] 用于维护每个进程号对应到进程信息结构体数组 proc[] 的哪个位置。 proc 结构体中维护了一个当前进程的所有孩子的链表，孩子数量，进程名等信息。由于系统中真正在运行的进程个数远小于 $2^{22}$，这个做法可以省很多内存。

事实上，Linux 的 pstree 工具还会将一个进程下的所有线程打印出来。我们可以在 `/proc/[pid]/task` 下看到所有的线程文件夹，`/proc/[pid]/task/[tid]/stat` 中也有各个线程的状态，不过本实验不要求打印线程。

## Print tree

打印进程树可以用对进程树的一次 dfs 来实现 —— 根节点是 1 号进程 systemd。这里需要注意的是遍历顺序的问题：在默认情况下，pstree 各个子树会按照进程名的字典序排序，如果有 `-n` 选项，则要按照进程号从小到大排序。在笔者的实现中，单向链表不方便进行高级的排序，但我们可以直接交换两个链表内部的内容，使用朴素的冒泡排序来完成工作。

> **更优美的显示**
>
> 打印树结构的最简单的方法是使用缩进：我们只需要记录当前递归到第几层，并打印相应个数的 TAB 即可。为了实现像 Linux pstree 工具那样精美的树结构，我们需要以下观察：
>
> * 每个进程的第一个子进程不换行。仔细思考以下，我们会发现这个规则等价于我们只会在那些叶子进程处换行。
> * 每个进程名之前有一个树的节点符号，这个节点符号遵循的规则如下：
>     * 如果某进程有且仅有一个子进程，则节点符号为 `-`；
>     * 如果某进程有大于一个子进程，则第一个孩子的节点符号是 `+`，最后一个孩子的节点符号是 <code>&#96;</code> ，中间孩子的节点符号是 `|`。
> * 当我们在打印内层进程内容时，外层进程树会有 `|` 延伸下来，我们可以用一个栈记录所有需要打印 `|` 的位置。但需要注意的是：每个进程的最后一个子进程如果还有子进程，子进程的子进程是不会被子进程的父进程的 `|` 包裹住的，具体操作上，每个进程的最后一个子进程负责将父进程从栈中弹出。
