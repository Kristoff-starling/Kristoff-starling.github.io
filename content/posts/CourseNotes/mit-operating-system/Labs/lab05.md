---
title: "MIT-6.S081 Lab 05: Lazy allocation"
linktitle: 'Lab lazy'
type: docs
draft: false
featured: false

math: true

menu:
    mitos-labs:
        parent: Contents
        weight: 5
---

## Progress

- [x] Eliminate allocation from sbrk() (easy)
- [x] Lazy allocation (moderate)
- [x] Lazytests and Usertests (moderate)

## Eliminate allocation from sbrk() (easy)

该部分要做的事情很简单：在 `sbrk()` 系统调用的处理函数中，不要分配内存，只修改进程的 sz 即可。由于我们只修改了 sz 而没有为新申请的空间绑定页面，所以当用户程序使用这块内存的时候 (load/store指令)，就会因为页表无法翻译虚拟地址而报 kernel panic。

## Lazy allocation (moderate)

在该部分需要做的修改和注意事项在课上基本讲过，可以参考 Lecture 08 的课堂笔记。总体来说，我们需要在 usertrap() 中识别出缺页异常并写一个缺页异常的处理函数：申请一个物理页面，将导致错误的虚拟地址所在的虚拟页面映射过去。此外，由于延迟分配的存在，我们在 uvmunmap() 时释放的不一定是已经映射过的地址，应当取消其中的一些 panic 语句。

## Lazytests and Usertests (moderate)

在该部分我们要额外处理 `sbrk()` 系统调用的一些细节：

* 当 `sbrk()` 传入的参数是负数时，我们应当调用 uvmdealloc() 正常进行释放。uvmdealloc() 释放的虚拟页面可能已经映射了物理页面，可能还没有映射物理页面。我们在上一个部分中已经取消了 uvmunmap() 函数中对该内容的检查，所以没有问题。

* 当 `sbrk()` 进行了非法的操作，如引起错误的虚拟地址不在堆区范围内，或者内存不足、无法分配新物理页面，则杀死当前进程。

* 除了缺页异常中需要处理延迟分配的问题，`read()` / `write()` 系统调用中也要处理延迟分配问题。该系统调用的函数调用路径 (以 write() 为例) 为  sys_write() $\rightarrow$ filewrite() $\rightarrow$ writei() $\rightarrow$ either_copyin() $\rightarrow$ copyin() $\rightarrow$  walkaddr()。内核为了在内核空间和用户空间之间传递数据，会调用 copyin()/copyout()/copyinstr()。这些函数会抓着用户虚拟地址在用户页表中进行一次 page table walk，以翻译出其物理地址。该过程中可能会因为延迟分配的问题翻译失败，我们要判断这种情况，并进行分配，代码细节和 usertrap() 中的非常类似，但需要注意的是由于 usertests 的设置问题，<u>在 walkaddr() 中遇到非法情况不能直接杀死进程</u>。

    (注：事实上，如果将 lab pgtbl 中的代码添加进来，用户进程和内核共用一份页表，就可以由硬件完成所有的 page table walk，延迟分配的处理函数也只需要写一遍。)

该实验中还有如下的几个细节：

* 笔者查阅了一些网上的代码，发现它们处理 “用户栈地址” 的方式各不相同：有的人使用 p->kernel_sp，这实际上是进程的内核栈的栈顶地址。有的人使用 p->trapframe->sp，这是用户进程在陷入内核时的用户栈指针，由于用户栈此时不一定空，所以 p->trapframe->sp 并不一定指向栈底。从严谨的角度来说，获取栈底地址应该使用 `PGROUNDUP(p->trapframe->sp)`。笔者采用的另一种稍愚笨一些的方法为：在 proc 结构体中新建一个 ustack 变量，在 `exec()` 系统调用中把用户栈底的地址保存在其中。
* 该实验中对于边界情况的判断要格外小心，usertest 中有很多针对边界情况的测试。p->sz，即 program break，指向的是用户内存顶部的下一个地址，第一个未被使用的地址，因此判断合法范围时一定是 `va < p->sz`。此外 `PGROUNDUP(p->trapframe->sp)` 得到的地址为用户栈底地址的下一位，因此下界一定是 `PGROUNDUP(p->trapframe->sp) <= va` 。(usertest 中的 copyinstr3 测试了当参数字符串的结束符处于未分配空间时程序是否能返回 -1)

