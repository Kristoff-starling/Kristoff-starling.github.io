---
title: "Lab 03: User Level Processes"
linktitle: 'Lab 03: uproc'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

math: true

menu:
    njuos-labs:
        parent: Contents
        weight: 9
---

## Overview

该实验的主要内容是实现用户进程。用户进程可以理解为套上了虚拟内存的线程，因此我们需要引入 VME 系列的 API。本实验报告主要记录一些有趣的设计和遇到的 bug。

## Designation

### ucontext()

由于笔者在 L2 中选择了通过汇编执行栈切换和执行流切换的操作，因此在 L3 的开始要做一些额外的改进。在 L2 中没有虚拟内存和页表切换的概念，因此第一次进入一个线程的时候我们可以很容易地通过一个內联汇编的 wrapper call 直接跳转到线程代码执行。但引入了用户进程后，初始上下文结构体的创建变得比较困难，返回用户态也需要做更多的事情 (比如页表切换，弹栈等)，因此笔者希望使用 AM 的 API。这就产生了一些矛盾。

幸运的是，笔者发现 AM 中 `trap64.S` 中的汇编函数 `__am_iret()` 是有 label 的，这意味着笔者只需要抄写一些 AM 的头文件，就可以调用 `__am_iret()` 来完成用户态的返回。 笔者最终的代码是这样权衡的：

* 因为调度器 swtch() 的原理是恢复内核线程执行的上下文，所以我们仍然需要一个类似于 xv6 中 forkret() 函数那样的东西作为一个进程第一次进入用户态的跳板。创建进程时，我们需要伪造一个内核线程的执行现场来让调度器跳转到一个函数 task_start()。
* 在 task_start() 中，我们需要完成 AM 中从 user_handler() 返回到调用 `__am_iret()` 之间做的事情，即切换用户页表和栈指针。然后调用 `__am_iret()` 完成最后的工作即可 (弹栈+`iretq` 返回用户态)。

### Page Table Walk

VME 对外的接口没有 page table walk，这让进程的复制变得困难：我们无法创建出一个和父进程页表一模一样的子进程页表。一种选择是在内核代码中对着 `vme.c` 中的 ptwalk() 写一个功能类似的函数，但笔者使用了比较简单的实现：在每个进程中额外维护了一个链表，记录了该进程拥有的所有虚拟页面以及其对应的物理地址和保护参数。链表这样扁平的结构会使得将来的查找效率变低，但比较容易写对且比较简单。

### wait()/exit()

xv6 book 中花了大量篇幅讲解这两个 API，因为这里处理稍有不慎就可能导致 data race 或死锁。在 oslab 中该问题的难度大幅下降，因为我们不需要在 exit() 时将孤儿进程 reparent 给 init，但仍需要非常小心。

笔者在书写代码时注意了锁的全局拓扑序问题：笔者规定任意一处代码如果要同时获得两个进程的锁，必须先获得父进程的锁再获得子进程的锁，这样一定程度上避免了死锁风险。

### Copy-on-write Fork

实现 cow fork 的基本思路为：在父进程 fork 时，将页面的写入权限去掉，并让父子进程的页表指向同一个物理页面。将来如果某个进程对该页面写入，则会触发 page fault，在 page fault handler 中我们新申请一个页面，将原页面的内容复制一份，修改当前进程的页表使其指向新页面，并重新赋予其写入权限。

cow fork 会带来一些额外的细节，比如我们需要对每个页面维护一个 reference count 表示当前有多少个进程使用这个页面。在释放一个进程时我们不能直接释放它所有的物理页面，而是要修改其 reference count，只有一个页面的 reference count 变为 0 时才能释放。笔者的设计是将 reference count 的增加工作放在 fork() 和 pgmap() 中，将 reference count 的减少工作放在 kfree() 中，由 kfree() 判断当前减完 rfcnt 后是否需要执行真正的 free()，这样设计的好处在于之前写的涉及 free 的代码不需要修改。

需要格外注意的是在引入 cow fork 后，一个页面就不只是一个进程的“资产”了，在处理页面 metadata 相关的信息时要记得上锁。

### mmap()

用户进程的虚拟地址空间理应以页为原子单位，因此笔者将 mmap() 参数中的 length 向 pgsize 上取整。分配时我们要确定一个地址 addr 是否满足 `[addr, addr + length)` 与当前地址空间没有冲突，由于无法直接访问页表，该检查笔者通过之前提到的链表完成，这一部分效率比较低下。

mmap() 要注意 MAP_PRIVATE 和 MAP_SHARED 两种页面的性质差异。对于前者，fork 时要进行 copy-on-write 的处理，去掉写入权限并在 page fault handler 中进一步处理；对于后者，fork 时不需要做改动。父子进程对于 MAP_SHARED 的页面的读写的安全性由用户自己保证。

## Interesting Bugs

### dfs-fork()

笔者在移植 dfs-fork 时遇到了用户态指针跑飞的情况。经过检查发现跑飞的地址就是 `dfs-fork.c` 中 map[] 数组的地址。经过仔细检查，笔者发现 Makefile 中的命令

```bash
x86_64-linux-gnu-ld -static --omagic --pic-executable --no-dynamic-linker --gc-sections -o _init -e _start trampoline.o ./printf.o ./init.o
x86_64-linux-gnu-objcopy -S -j .text* -j.rodata* -j .data* -j .bss* --set-section-flags .bss=alloc,contents -O binary _init
```

前者并不能保证整个程序静态链接，map[] 的地址被放在了 GOT 中。在后续的 objcopy 中，只有代码节、数据节、.bss 节被保留了下来，从而 map[] 的地址丢失了。

在 dfs-fork 的全局数组前加上 static 修饰就可以将数组放到数据节中，从而解决这个问题。

### Page Table Teardown

笔者在移植 dfs-fork 进行多核测试时曾经遇到神秘的 CPU reset。经过排查发现这涉及一个比较微妙的页表问题。假设一个核上父进程正在 wait()，另一个核上子进程在 exit()。根据笔者的设计，子进程 exit() 后会唤醒正在等待的父进程，然后跳转到调度器线程。但需要注意的是调度器线程现在使用的仍然是“子进程”的页表。父进程醒来后发现有僵尸子进程，于是回收资源，回收时调用 unprotect() 销毁子进程的页表，从而导致另一个核的调度器线程 crash。

如果在 L2 中没有使用內联汇编做栈切换，我们应该会有一个“等待下一次进入 os_trap 再释放上一次进入 os_trap 的进程的锁” 的操作，这个操作在 L3 中同时可以确保页表不会被提前 teardown。但笔者的 L2 设计的特殊性导致了这里必须处理这样一个相似的问题。笔者的解决方法非常简单：用户进程在陷入内核后执行的都是内核代码，因此我们可以在 os_trap 的开头把页表切换成内核页表，这样就不会 crash 了。

