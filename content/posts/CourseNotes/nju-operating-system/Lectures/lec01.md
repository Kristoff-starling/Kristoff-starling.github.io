---
title: "Lecture 01: Introduction"
linktitle: '01: Introduction'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 1
---

> **关于计算机信息**
>
> 使用 `uname -a` 命令可以查看操作系统，时间，硬件架构等。

<!-- more -->

## What is Operating System?

> Operating system is responsible for makeing it easy to run programs, allowing programs to share memory and interact with devices.  - OSTEP

操作系统：“管理软硬件资源，为程序提供服务”的东西。

### ENIAC

* 逻辑门：真空电子管。

* 存储器：延迟线内存，类似于“小丑扔球”的技巧，将数据不断地扔进延迟线，拿出来以后经过放大器再扔进延迟线。

* 输入输出：打一针/将纸带挪动一下。

不需要操作系统。能运行程序就不错了，不需要“管理程序的程序”。

### 1950s Computer

更快更小的逻辑门 (晶体管)，更大的内存 (磁芯内存)，更丰富的 I/O 设备。因为 I/O 设备速度严重低于处理器速度，所以中断机制出现。更多的人使用通用计算机，他们希望使用 API，而不是直接使用硬件。因此诞生了 FORTRAN 编程语言。

1950s 的操作系统：管理多个程序依次排队运行的库函数和调度器。

* 计算机非常贵非常少，多个用户排队使用计算机。
    * 很多程序卡片 $\rightarrow$ OS $\rightarrow$ 硬件
* 操作 (operate) 任务 (job) 的系统 (system)
    * 批处理系统：程序自动切换 (换卡) + 提供库函数 API

### 1960s Computer

更大的内存，更多的高级语言 (COBOL, BASIC etc.) ... 我们可以将多个程序同时放到程序里。但计算机中只有一个 CPU。

1960s 的操作系统：

* 传统的执行流：A on CPU (30s) $\rightarrow$ A on device (5min) $\rightarrow$ A on CPU (30s) B on CPU (30s)

* 理想的执行流：

    CPU: 		A on CPU(30s) $\rightarrow$ OS 调度 $\rightarrow$ B on CPU (30s) 		$\rightarrow$ OS 调度 $\rightarrow$ A on CPU

    Device:						                       	$\rightarrow$ A on device (5min)

这就是多道程序 (multiprogramming)。为了防止恶意程序对别的程序和操作系统本身的破坏，我们还需要对每个程序的运行有一个好的隔离机制。

多道程序在程序使用设备时发生切换。在引入时钟中断后，操作系统成为计算机的霸主：操作系统决定谁来运行。

### 1970s+ Computer

计算机空前发展，与今天已无大异。

分时系统趋于成熟，UNIX 诞生，奠定现代操作系统的形态。

### Today

操作系统：虚拟化硬件资源，为应用程序提供服务。

面对的问题：

* 更复杂的处理器和内存：非对称多处理器， Non-uniform Memory Access etc.。
* 更多的设备和资源。
* ……

