---
title: "Lab 02: Kernel Multi-Threads"
linktitle: 'Lab 02: kmt'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

math: true

menu:
    njuos-labs:
        parent: Contents
        weight: 8
---

## Overview

该实验的主要内容是实现内核线程的调度。本实验报告主要记录一些有趣的设计和测试方法。

## Designation

### spinlock

在 L2 中我们需要实现一个相较于 L1 更加精细的自旋锁。在引入了中断机制后，我们在临界区内需要关闭中断，否则一旦持有锁的线程被调度下了 CPU 一整个时间片就会卡住。我们需要小心地维护中断的嵌套。我们需要一个 per-cpu 的变量来记录中断嵌套的层数，在第一层的时候关闭中断，并在最后一层被解开时打开中断。

### Inline Assembly

笔者的初代实现使用了內联汇编进行栈切换 (模仿 xv6 的设计)，因而是架构相关的 (笔者针对 x86-64 和 x86 各写了一遍汇编)。在这种实现下，一次线程切换 (以时钟中断为例) 会经历如下历程：

* 经过 AM 中的一些代码后，上下文保存在了栈上的 Context 结构体中，进入 os->trap。
* os->trap 根据中断原因，进入时钟中断的 handler。时钟中断要求当前线程主动让出 CPU。handler 会通过一个 swtch() 函数 (內联汇编) 跳转到 scheduler 线程，该函数原理类似 setjmp 和 longjmp，恢复寄存器现场并切换栈。scheduler 线程使用自己的独立的栈 (即不是线程的内核栈)，每个 CPU 核一个。

* 调度器线程是一个死循环，它不断选择状态为 RUNNABLE 的线程并调用 swtch() 函数切换过去。
* 目标线程返回自己的 Context 结构体。

这种写法下，每个 os->trap 最终返回的 Context 结构体就是传进来的那个，因此不需要对 Context 做额外的保存。这其中有一些细节需要注意：

* scheduler 线程必须打开中断，比如当前所有 input_task 线程都处于睡眠状态，我们只有让调度器线程响应键盘中断信号，才能唤醒这些 input_task 线程。
* CPU 上锁之前的中断开关情况是一个 proc local 的状态，在调用 swtch() 切换前必须保存下来：比如在笔者的实现中，我们可能是在中断处理程序中 yield，这时 intena 是关中断状态；而信号量的 sleep() 中也会调用 swtch()，这时 intena 是开中断状态。
* 不使用 AM API 的最大坏处是：我们必须手动维护第一次进入一个线程的过程，因为我们之后切换到一个线程依赖的是它之前有一个 swtch() 到 scheduler 的现场，但一个进程刚被创建时没有现场。笔者的解决方法是：伪造一个寄存器现场完成栈切换，并通过一个內联汇编的 wrapper call 直接跳转到目标函数执行 (类似 M2)。

### Semaphore

笔者为每个信号量准备了一把锁，这样可以用类似条件变量的方式实现信号量。sem_wait() 和 sem_post() 共用信号量的锁，sleep() 通过线程的锁来保证原子性。这其中的关键细节在于 sleep() 中先获取线程锁再释放信号量锁，这样对于 wait() 函数来说，sleep() 释放锁、线程睡眠 (修改线程的状态) 和线程切换这几个步骤就是原子的。

有了信号量之后，我们可以使用 binary semaphore 作为一个睡眠锁，在长临界区的代码片段中睡眠锁可以提升效率。

## Testing

### Producer Consumer

在多核上多线程跑生产者消费者是一种比较好的暴露并发 bug 的好方法。在测试集的参数上笔者有如下心得：

* 我们并不需要将括号打印在屏幕上肉眼检查，可以选择写一个 python 程序接收 oslab 的输出自动检查，或者在 oslab 的括号输出函数中直接修改一个全局计数器并执行检查。
* 创建较多的线程，将括号的嵌套层数调得浅一些 (比如 5 以内) 比较有利于测试并发正确性，因为嵌套层数浅时更容易触发错误，以及生产者/消费者的睡眠条件更容易满足，程序的并发行更高。
* 创建较小的线程可能有利于检查死锁错误。

### dev Test Suite

一次完整的输入输出会经过如下步骤：

* `tty_reader` 线程向 tty 输出命令行提示符，然后调用 tty 的 read() 函数。tty 的 read() 函数要等待 tty_cook() 的信号，因此暂时挂起。

* 每当用户按下一个键，硬件会生成一个中断，dev 测试集中的 input_notify() 函数负责处理这个中断，它会发送一个 kbdirq 信号，唤醒正在睡眠的 `input_task` 线程。`input_task` 线程从设备寄存器中读取按键的内容，更新当前的键盘按键状态，调用 push_event() 将按键事件放入 `in->events[]` 队列中，并发送一个 event_sem 信号。此外，时钟中断也会发送 kbdirq 信号唤醒 `input_task` 线程，如果一段时间内没有任何按键，`input_task` 会 push 一个空的按键事件并发送 event_sem，向别的线程提供一个伪时钟。

* `tty_task` 线程会不断调用 input_read() 函数读取键盘输入，input_read() 会调用 pop_event() 从 `in->events[]` 队列中取出一个按键事件返回，对于一个普通的按键事件，`tty_task` 会调用 tty_cook() 函数将字符加入一个和读取线程共享的队列 (保护该队列的锁是 `tty->lock`)，并调用 `ttydev->ops->write` 将字符回显在 tty 上。

    tty_cook() 调用 tty_enqueue() 函数将字符放入队列 `tty->queue`，如果遇到换行符 `\n`，tty_cook() 会发送 cooked 信号来唤醒 `tty_reader` 线程。

* tty_read() 函数被唤醒后，会将队列里的字符拷贝到指定的地址。然后 `tty_reader` 线程打印相应的长度信息。

注：tty_read() 打印信息和字符回显的先后无法保证。这里理应有的顺序是：在字符回显完成后 `tty_reader` 被唤醒并打印信息；`tty_reader` 打印出下一个命令行提示符之后再进行字符回显示。如果要获得更好的体验可能需要一些同步机制。

