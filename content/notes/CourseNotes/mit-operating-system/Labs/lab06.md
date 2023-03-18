---
title: "MIT-6.S081 Lab 06: Copy-on-write fork"
linktitle: 'Lab cow'
type: docs
draft: false
featured: false

math: true

menu:
    mitos-labs:
        parent: Contents
        weight: 6
---

## Progress

- [x] Implement copy-on write (hard)

## Implement Copy-on Write

写时拷贝 fork 的大体思路是：在 fork 时不为子进程创建新的页面，而是让子进程指向父进程的所有页面，并将页面的权限改为只读。将来如果某个进程 (父/子) 需要修改页面，会发生 page fault exception，在 kernel 的处理程序中，如果是 COW page，则新分配一个页面，将当前的只读页面内容复制过去，让当前进程的页表指向这个新的页面并将权限设置为可读可写。

### kalloc()/kfree()

引入了 copy-on write 之后，一个物理页面可能会被多个进程的页表指向，这使得 kfree() 变得困难：某个进程被释放时，它拥有的物理页面不能被直接释放，否则别的进程就用不了这个页面了。我们需要为每个页面维护一个 reference count，只有一个页面的 reference count 为 0 了才能释放它。

维护 reference count 的时机很有讲究。笔者最初使用的方案是在“映射“和“解除映射”的时候修改 reference count，即在 mappages() 函数、 uvmunmap() 函数和 COW 相关函数中修改 reference count。但 xv6 中对 mappages() 的用法比较灵活，例如

```c
void
kvminit()
{
  ......
  // map kernel data and the physical RAM we'll make use of.
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  ......
}
```

上述 kvmmap() 会对所有的物理页面做一个恒等映射。其实内核并没有真正拥有这些页面，做 mappages() 时，真实分配好的页面是不存在的，在 kvmmap() 中做这一步只是为了简化内核代码的书写 (如在调用 memmove() 时不需要在软件中跑一遍内核页表)。因此如果在 mappages() 的时候维护 reference count，会得到不正确的结果。

一个合理的解决方案是在 COW 相关函数中增加 reference count，在 kfree() 中减少 reference count。kalloc() 中新出厂的页面默认 reference count 为 1。

### uvmcopy()/uvmcow()

我们需要修改 uvmcopy() 的实现，不为子进程分配新的页面，而是让子进程指向父进程的页面，更新父进程的 reference count，并设置页表项的标志位。这里要注意的是，除了取消页面的 PTE_W，为了将 cow page 的异常和普通的越权修改异常区分开来，我们应当在页表项中标志一个 PTE_COW 位。RISC-V ISA 的页表项格式中，8，9,10 三位是 RSW 位，留给软件自由使用，我们可以利用这些位来标记 PTE_COW。

uvmcow() 是 COW 导致的 page fault exception 的处理函数。我们应当分配一个新页面，新页面没有 PTE_COW 位，有 PTE_W 位，将原始页面的内容复制过来，并修改当前进程的页表。

### Concurrency

该实验中我们要开始面对并发 bug。`/kernel/kalloc.c` 中的空闲页面链表是用 kmem.lock 保护起来的。与此同理，我们维护 reference count 的数组也应该用一把锁保护起来。我们当然可以使用 kmem.lock，但如果新建一把锁专门用于保护 reference count 数组 (甚至是每个 referenct count 的单元格) 可以获得更好的并行度。

Concurrency bug 中常见的一种是 TOCTTOU (time to check to time to use)。笔者曾经遇到过 xv6 在单核情况下可以正常运行，在多核情况下内存不够用的问题。经过排查发现问题出在如下代码：

```c
if (krref(pa) == 1)
{
    *pte |= PTE_W;
    *pte &= (~PTE_COW);
}
else
{
  if ((mem = kalloc()) == 0) return -2;
  flags &= (~PTE_COW); flags |= (PTE_W);
  memmove(mem, (void *)pa, PGSIZE);
  kmref(pa, -1);
  if (mappages(pagetable, va, PGSIZE, (uint64)mem, flags) != 0) return -1;
}
```

这段代码的逻辑是：如果当前页面只剩一个 reference，就不必再申请新页面，而是直接修改一下本页面页表项的标志信息。否则当前页面有多个 reference，申请一个新页面并让当前进程指向它，调用 `kmref(pa,-1)` 更新原页面的 reference count。

这段代码的 bug 在于进入 else 分支后，`krref(pa)>=2` 的假设不一定成立。假设有两个进程并发地执行该程序段且 pa 有且仅有两个 reference，那么两个进程都会通过 if 判断进入 else 分支，然后使用 kmref() 更新 reference count，这样 pa 页面的 reference count 变成了 0 但没有进程去 free 它，我们丢失了一个页面。这个 bug 触发次数越多，系统内可用的内存就越少。

想要修复这个 bug，一种选择是给整个 if 语句上锁，另一种选择是不使用 if 并把 kmref() 改成 kfree() 或 uvmunmap()。