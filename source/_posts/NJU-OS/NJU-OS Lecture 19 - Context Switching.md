---
layout: post
date: 2022-04-25
title: "NJU-OS Lecture 19: Context Switching"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

如果我们的用户程序有死循环，计算机并不会被卡死——因为操作系统有进程调度；但操作系统有对整个计算机完全的掌控权，如果在操作系统代码中加一个死循环，那计算机就真的卡死了。<!-- more -->

## Virtualization of CPUs 

每一个进程都是一个状态机，其中有寄存器、内存，看内存的 "VR 眼镜" (页表) 等等。操作系统是一个状态机的管理者：它要维护各个进程的状态机，此外内核对应的状态机也归 OS 管理。正在运行的状态机的状态保存在硬件上 (CPU, DRAM)，其他未运行的进程的状态机以某种方式被暂存起来。

现代操作系统通常借助时钟中断，使用抢占式多任务的方式来在状态机之间切换。中断相当于一个强行插入的 ecall，打断正在运行的执行流。操作系统负责将这个进程当前的状态机从硬件上复制下来，挑选下一个准备执行的进程，然后将其状态机搬上硬件。

## Code: Xv6 trapframe & Thread Switching

该部分内容可以参考 MIT 6.S081 Lecture 6 和 Lecture 11。