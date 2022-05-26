---
layout: post
date: 2022-02-47
title: "OSTEP Chapter 05: Interlude: Process API"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

## 5.1 The `fork()` System Call

fork() 系统调用会创建一个和当前进程完全相同的子进程，这两个进程除了 fork() 的返回值不同，其他完全相同。fork() 在父进程中返回子进程的进程号，在子进程中返回 0，这可以用于区分两个进程。

[p1.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2005%20-%20cpu_api/p1.c) 展示了一个使用 fork() 的例子，其中 getpid() 函数可以获得当前进程的进程号。注意这个程序的运行结果是 non-deterministic 的：父进程和子进程谁会先输出，这取决于 OS 的调度器。现代操作系统的调度策略极其复杂，可以看[这篇文章](https://linux.cn/article-7325-1.html)有一个大概的了解。<!-- more -->

## 5.2 The `wait()` System Call

[p2.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2005%20-%20cpu_api/p2.c) 展示了一个使用 wait() 的例子，wait() 提供一种同步机制，保证了父进程在子进程结束之后再执行。如果父进程先被调度器选中，那么它执行 wait() 会被阻塞，直到子进程结束；如果子进程先被调度器选中，那么父进程执行 wait() 时看到子进程已经退出则会立刻返回。

> **Zombie Process**
>
> 一个子进程如果已经退出但还没有被它的父进程 wait()，那它就是一个僵尸进程。内核保存了僵尸进程的一份 minimal 的信息，包括进程号、退出状态等，这样以后如果父进程 wait() 了，父进程就可以获取子进程的信息。
>
> 被 wait() 过的僵尸进程会被从 process table 中移除。一个 zombie process 如果不被 wait，就会一直待在 process table 中，一旦内核的 process table 满了，就不能再创建新的进程。因此父进程应该及时清理自己的僵尸子进程。如果父进程退出了，那么它的僵尸子进程会被 init 进程 (或其他某个指定的进程) wait 掉。

## 5.3 Finally, The `exec()` System Call

[p3.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2005%20-%20cpu_api/p3.c) 展示了一个使用 exec() 的例子。exec() 传入一个可执行程序的文件名，它会将该文件的代码和数据加载到内存中覆盖当前的代码和数据，重新初始化堆区和栈，并跳转到新程序的入口开始执行。exec() 不创建新的进程，它只是重启当前的状态机。exec() 如果执行成功就不会返回。

## 5.4 Why? Motivating the API

将 fork() 和 exec() 分开的设计似乎有一些反人类：我们创建一个新进程运行新程序需要两个系统调用。这样设计的真正目的是让我们有机会在 fork() 和 exec() 之间做一些有意思的事情。

[p4.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2005%20-%20cpu_api/p4.c) 展示了一个重定向的例子。我们在 fork() 之后 exec() 之前关闭 stdout，再打开一个新的文件，这样新打开的文件的描述符就是 1 (stdout)，这是再 exec()，我们就实现了将标准输出重定向到文件。除了重定向，UNIX 的管道机制也是通过在 fork() 和 exec() 中间操作实现的。

## 5.5 Process Control And Users

除了 fork(), wait() 和 exec()，UNIX 系统中还有很多其他控制进程的 API，比如 kill() 用于给进程发送信号。信号机制可以向进程通知外部事件的发生，常见的信号有 SIGINT (interrupt，通常用于结束程序)，SIGTSTP (暂时挂起程序)，SIGSEGV (段错误) 等。

既然信号的能力这么强，那么肯定不能让任意用户随便给别人的进程发送 SIGINT。你需要通过密码登录获取管理员权限，或者你本身是 root 用户，才能执行这些系统资源相关的操作。在用户态，你只能管理你自己的进程。

## 5.6 Useful Tools

ps 命令可以查看当前活跃的进程。top 命令可以查看所有进程的详细信息。

## 5.7 Summary

略。