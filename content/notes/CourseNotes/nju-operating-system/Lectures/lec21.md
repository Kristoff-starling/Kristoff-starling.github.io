---
title: "Lecture 21: OS Design"
linktitle: '21: OS设计'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 21
---

操作系统的设计：一组对象+访问对象的 API；操作系统的实现：一个 C 程序完成上面的设计

我们关心的问题是：操作系统到底应该提供什么样的对象和 API？

## Monolithic Kernel

Unix 系列：[The Open Group Base Specifications Issue 7 (2018 Ed.)](https://pubs.opengroup.org/onlinepubs/9699919799/mindex.html)

Windows 系列：[Windows API Index](https://docs.microsoft.com/en-us/windows/win32/apiindex/windows-api-list)

不同的 API 系列可以互相模拟，于是有了 WSL 和 WINE。

> **Real Operating Systems**
>
> 真正的操作系统要考虑的事情总是比想象中的复杂。比如我们写一个简单的重命名函数：
>
> ```c
> void rename_file(char *oldname, char *newname) {
>  char buf[SIZE];
>  int fd = open(oldname, O_RONLY);
>  fread(fd, buf);
>  close(fd); delete(oldname);
>  // <------------------
>  fd = open(newname, O_CREAT | O_TRUNC | O_WONLY);
>  write(fd, buf, sizeof(buf));
> }
> ```
>
> 如果在箭头指向的时刻操作系统突然掉电了，那么文件内容就全部丢失了！这显然是不合理的现象。阅读 rename() 系统调用的手册，可以看到手册保证了删除旧文件，创建新文件操作的原子性。

## Microkernel

复杂系统的正确性难以保证。像 Linux 这样以 C 为主要语言的操作系统，C 的 undefined behavior 会给系统带来无穷的灾难：比如文件系统一旦出错，整个内核中的任何模块都可能受到损坏。

微内核的想法是：将尽可能多的功能用普通进程实现，将问题隔离在进程级。比如文件操作，UNIX 的 write() 会陷入内核，但另一种想法是：调用一个 remote_write() 和一个 File server 交互，FS进程有访问磁盘的权限即可。

一些好的想法：

* 一些和状态机管理密切相关的系统调用没法被搬出内核：比如 fork(), mmap()。
* 如果操作系统提供一个 mmio() 系统调用，让进程通过访问内存的方式来控制设备寄存器、磁盘 etc，那么设备驱动代码，文件系统代码都可以放在用户态。

赋予进程最少的权限，就能降低错误带来的影响。

### Minix

Minix 是一个具有跨时代意义的教学操作系统，它采取的是微内核设计。在 Minix 2.0 中，内核只提供了两个 API：send 和 receive。基于这两个系统调用，我们实现一些 RPC (remote procedure call) 来实现进程之间的通信。Kernel 里只有很少的状态机相关的机制：比如中断，异常，时钟驱动，内存映射 etc. 其他的功能，比如文件系统，设备驱动，网络 etc. 都在用户态。跨模块的调用会跨越进程边界 i.e. 地址空间的切换等，这样一个模块的错误就可以被隔离在本地，UB 不会波及到模块外部。

### seL4

seL4 是第一个完成了正确性证明、运行时间上界证明的微内核。

seL4 的证明思路大致如下：以 `thread-os.c` 的 round-robin 调度器为例，首先我们用适合描述行为的语言建立一个模型 (这里以 python 为例, seL4 在这里实际上有两层建模)：

```python
def rr_sched(cpu):
    cpu.threads = cpu.threads[1:] + cpu.threads[:1]
    assert anything_you_need
    return cpu.threads[0]
```

我们可以用 model checker 确认这个 high level 语言编写的程序的正确性，然后我们希望证明 `thread-os.c` 和 python 代码的行为等价性。除去访问硬件的部分 (AbstractMachine)，操作系统本质上是纯计算的程序 (数学模型)，因此我们只需要观测所有的 AM call 行为是否等价即可。如果我们使用被 verified 的编译器，我们还可以进一步证明 C 程序和汇编指令的等价性。

## UniKernel

很早的时候就有了 exokernel (外核) 的概念，即操作系统不应该有任何的策略，只需要提供最小的硬件抽象。在有了虚拟机的时代，这催生了 unikernel：将操作系统的内核代码和应用程序直接链接起来跑，因为硬件都是虚拟机抽象出来的，所以没有安全问题。这时的 OS 类似于一个 libOS 的功能。