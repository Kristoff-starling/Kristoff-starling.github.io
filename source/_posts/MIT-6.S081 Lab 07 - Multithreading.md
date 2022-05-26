---
layout: post
date: 2022-04-24
title: "MIT-6.S081 Lab 07: Multithreading"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lab Reports"]
tags: 
mathjax: true
---

## Progress

- [x] Uthread: switching between threads (moderate)
- [x] Using threads (moderate)
- [x] Barrier (moderate)

<!-- more -->

## Uthread: Switching Between Threads (moderate)

该实验要实现一个“用户态协程库”，每个协程通过 yield() 函数主动让出 CPU 来让其他协程运行。笔者定义了如下的结构体：

```c
struct thread {
  char            stack[STACK_SIZE]; /* the thread's stack */
  int             state;             /* FREE, RUNNING, RUNNABLE */
  struct context *context;
};
```

其中 context 指针指向的地方用于存储和恢复寄存器现场，笔者的代码会利用堆栈低地址处的 `sizeof(struct context)` 的空间来存放这个 context。

### thread_create()

类似于 xv6 中的设计，创建线程时我们要在 context 中伪造一个用于返回的寄存器现场：设置 ra 为目标函数地址，sp 为改线程的栈的地址即可。

笔者曾经遇到一个非常难以理解的 bug：某函数的代码在运行过程中被修改，导致段错误。经过排查发现：笔者忘了为 main() 函数对应的线程设置 context 指针，context 指针默认初始值为 0，从而在第一次上下文切换时，main() 的上下文覆盖了代码段。我们应该在 thread_init() 中为 main() 的线程做好初始化。

xv6 的用户进程中代码段是从 0 地址开始保存的，查看 `/kernel/exec.c` 可知 xv6 通过 uvmalloc() 申请页面并将代码放入，uvmalloc() 也是 sbrk() 系统调用底层的函数，申请的页面默认是 R/W/X/U 的，所以用户进程有修改的权利 (加载时将代码段设置为不可写或许更容易暴露 bug)。

### thread_switch()

这部分代码用汇编实现，可以完全参考 `/kernel/swtch.S` 的实现。类似地，因为 thread_switch() 在 C 代码中被当作一个普通函数调用，编译器会帮我们保证 caller-saved registers 的封存 (栈上)，我们只需要保存 callee-saved registers 即可。

### thread_schedue()

该函数负责挑选一个 RUNNABLE 的线程，然后通过 thread_switch() 跳过去。

### thread_yield()

该函数只需要将当前线程的状态改为 RUNNABLE，然后调用 thread_schedule() 即可。

## Using Threads (moderate)

该实验使用 Linux 的 pthread 线程库，是非常基础的带锁并发编程实验。为哈希表的每个 bucket 上一把锁就可以既保证正确性又通过速度测试。

## Barrier (moderate)

该实验使用 Linux 的 pthread 线程库，是非常基础的条件变量编程实验。我们甚至不需要生产者-消费者模型——我们只要让先来的线程在条件变量上睡眠，最后一个到来的线程 broadcast 一下即可。

一个小细节是我们要在 barrier() 中修改 round，但 round 只能被修改一次。我们可以让每个线程上来先保存一个旧的 round 值，然后被唤醒后每个线程上锁检查 round，只有看到 `old == new` 的线程可以 +1。