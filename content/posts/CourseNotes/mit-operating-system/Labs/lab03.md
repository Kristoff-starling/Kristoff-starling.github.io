---
title: "MIT-6.S081 Lab 03: Page tables"
linktitle: 'Lab pgtbl'
type: docs
draft: false
featured: false

math: true

menu:
    mitos-labs:
        parent: Contents
        weight: 3
---

## Progress

- [x] Print a page table (easy)
- [x] A kernel page table per process (hard)
- [x] Simplify `copyin/copyinstr` (hard)

## Print a page table (easy)

在 `/kernel/vm.c` 中新建一个函数 vmprint() 用于打印页表信息。为了方便编码，笔者在 `/kernel/vm.c` 中又定义了一个函数 vmprint_recursive()，该函数不仅接收页表地址，还接收一个 layer 参数，用于确定打印信息时在前面加多少组 `..`。vmprint_recursive() 遍历传入页表的 512 个页表项，如果页表项的 PTE_V 位为 1 (页表存在) ，则打印页表项的值和其对应的物理地址。如果当前页表项不是底层页表项 (叶子节点)，就递归地继续打印。我们可以根据页表项的标志信息来判断它是不是叶子节点：叶子节点的页表项除了 PTE_V 还会有其他的 RWXU 等权限位，而各级页目录页表项只有 PTE_V 为 1。

> Explain the output of `vmprint` in terms of Fig 3-4 from the text.  What does page 0 contain? What is in page 2? When running in user mode, could the process read/write the memory mapped by page 1?

第 0 个页面存储了用户的代码段和数据段，第 2 个页面是用户栈。在用户态下，进程不能读写第 1 个页面，因为根据第一个页面的页表项，其 PTE_U 位为 0，用户态无访问权限 (第 1 个页面是用户栈下方的保护页面，在创建后通过 uvmclear() 函数清除了 PTE_U 位)。

## A kernel page table per process (hard)

> 根据 RISC-V 架构的约定，page table walk 的工作理应交给硬件完成，那么为什么 xv6 中要有 walkaddr() 这样的函数来在软件层面进行 page table walk呢？

xv6 中当用户陷入内核态时，satp 寄存器会切换到内核页表。内核页表中只有内核部分的映射，因此在内核态中如果想要访问用户内存中的数据，就必须在软件层面进行虚拟地址的翻译：抓着用户页表和用户虚拟地址调用 walkaddr() 函数获得物理地址，再拿着物理地址进行操作 (内核页表是恒等映射，因此这个物理地址也可以当做 "虚拟地址" 使用)。

该实验部分和下一个实验部分的目的是：为每一个进程创建一个“用户内核页表”。该页表的内核部分映射与内核页表相同，用户部分映射与用户页表相同。这样用户进程从用户态陷入内核态时，不再切换到内核页表，而是切换到本进程的“用户内核页表”，这样在内核中执行操作时，页表中包含用户部分的映射，用户部分虚拟地址的翻译就可以交给硬件做了。

该实验部分主要负责完成用户内核页表的内核部分的映射。根据提示内容，

> Add a field to `struct proc` for the process's kernel page table.   

在 `struct proc` 中添加一个 pagetable_t 类型的变量 k_pagetable 来存储用户内核页表。

> A reasonable way to produce a kernel page table for a new process  is to implement a modified version of `kvminit` that makes  a new page table instead of modifying `kernel_pagetable`.  You'll want to call this function from `allocproc`.   

笔者起初认为：为什么不让每个进程的 k_pagetable 直接指向 kernel_pagetable 呢？这样就省去了为每个进程做映射的工作？如果理解了整个实验的目的，我们就会明白这是不合理的。我们的目的不仅是完成内核映射，还要让这份页表同时拥有该进程的用户映射，内核中的原始份内核页表无法将各个进程的用户映射都存下来 (彼此冲突)。这样做也许可以通过这个实验部分，但将在下一个实验部分走入死胡同。

正确的做法是仿照 kvminit() 函数写一份初始化映射，将页表基地址保存在 p->k_pagetable 中。allocproc() 函数在 userinit() 和 fork() 中被调用，负责创建一个新进程。我们在 allocproc() 中调用用户内核页表的初始化映射函数即可。

> Make sure that each process's kernel page table has a mapping  for that process's kernel stack. In unmodified xv6, all the  kernel stacks are set up in `procinit`. You will need to  move some or all of this functionality to `allocproc`.   

笔者没有动 procinit() 的代码，即内核页表中有所有进程内核栈的映射，并准备在每个进程的用户内核页表中映射自己进程的内核栈地址。这里笔者犯的错误是：在 allocproc() 中又 kalloc() 了一个新的页面作为用户栈。这样用户内核页表中的用户栈和内核页表中的用户栈实际上指向了不同的物理页面实体。我们要时刻记住**用户内核页表只应该复制映射，映射应该指向相同的实体**，这样陷入内核态之后才能看到页面中的改变。

> Modify `scheduler()` to load the process's kernel page table into  the core's `satp` register (see `kvminithart` for  inspiration). Don't forget to call `sfence_vma()`  after calling `w_satp()`.   
>
> `scheduler()` should use `kernel_pagetable` when  no process is running.   

这是笔者感到迷茫的部分，因为根据课程 schedule 的进度，做该实验时理应还没有看中断部分的内容，所以笔者对 scheduler() 函数的意义很不了解。经过大致的摸索，核心循环

```c
for(p = proc; p < &proc[NPROC]; p++) {
  acquire(&p->lock);
  if(p->state == RUNNABLE) {
    // Switch to chosen process.  It is the process's job
    // to release its lock and then reacquire it
    // before jumping back to us.
    p->state = RUNNING;
    c->proc = p;
    swtch(&c->context, &p->context);

    // Process is done running for now.
    // It should have changed its p->state before coming back.
    c->proc = 0;

    found = 1;
  }
  release(&p->lock);
}
```

负责找到第一个处于 RUNNABLE 状态的进程，然后将其状态设置为 RUNNING，并通过 swtch() 函数切换过去。也就是说，执行完 swtch 之后，xv6 就会转去执行进程 p (此时仍然在内核态中)，因此应该在 swtch() 之前将 satp 寄存器改写为 p 进程的用户内核页表。

一个进程运行完毕后，会返回到 swtch 的下一条语句执行，这时候 CPU 核处于没有进程运行的状态。因此应该在 swtch() 之后调用 kvminithart() 函数将 satp 寄存器改写为内核页表。

> Free a process's kernel page table in `freeproc`.  
>
> You'll need a way to free a page table without also freeing the leaf physical memory pages.   

freeproc() 函数负责进程的释放。其中 proc_freepagetable() 函数负责将该进程的用户页表释放。我们需要再写一个函数将用户内核页表也释放。注意用户内核页表的实体是那些内核物理页面和该进程的用户物理页面。该进程的用户物理页面已经随着用户页表的释放而释放了，而内核物理页面不能被释放，否则内核代码就消失了。因此我们不能直接调用 proc_freepagetable()，而要再写一个 proc_freekpagetable()。

此外，proc_freepagetable() 中调用的 freewalk() 函数负责递归地释放所有的页表页面，该函数默认底层的物理页面已经被释放，如果不符合要求会触发 kernel panic。但在用户内核页表页面的释放过程中底层的 (内核) 物理页面是没有被释放的，因此不能直接调用 freewalk() 函数，而要仿照它再写一个不会对底层物理页面存在报错的 kfreewalk()。

## Simplify `copyin/copyinstr` (hard)

该实验部分承接上一个实验部分，要补全用户内核页表的用户部分的映射。我们只要做到 xv6 中任何对用户页表有修改的地方，我们在用户内核页表中将这些修改的映射抄一份即可。这里务必要注意的一点是：**用户页表中会申请新的物理页面，但我们不应该在用户内核页表中也申请新的页面，而是应该映射到用户页表申请的物理页面**。只有这样我们才能在陷入内核态时访问到用户态时保存的数据。

xv6 涉及用户页表修改的有三处：fork()，exec()，和 growproc()。

* fork() 函数会申请一个新的进程，调用 uvmcopy() 函数将父进程的用户页表复制给子进程的用户页表。注意：我们不能调用 uvmcopy() 来将父进程的用户页表内容复制到子进程的用户内核页表。因为父进程和子进程互相独立，所以 uvmcopy() 在复制的时候不仅复制页表内容，所有底层的物理页面也申请了新的并复制了一遍 (在 copy-on-write fork 中这一点会得到改善)。我们的用户内核页表是不应该将底层物理页面复制的。**子进程的用户内核页表一定要抄子进程的用户页表，映射到子进程的用户页表的物理页面上**。

* exec() 函数根据 ELF 文件创建了一个新的内存镜像，并用新的页表替换旧的页表。在用户进程页表中我们也应该进行相对应的更新：首先将旧页表相关的映射删除 (这里需要格外小心的一点是：exec() 调用 proc_freepagetable() 释放旧页表时，已经将旧页表的底层物理页面一并释放了，我们在删除用户内核页表中的旧页表内容时，不需要再释放一遍底层物理页面，只需要清除相关的页表项即可。)，然后将新页表的映射抄进用户内核页表。

* growproc() 函数根据 n 的正负分别调用 uvmalloc() 和 uvmdealloc() 来调整用户内存的大小。我们在用户内核页表中也应该进行相对应的更新。对于 uvmalloc() 来说，要注意用户内核页表要映射到用户页表创建的那些物理页面中，为此笔者写了一个类似于 uvmcopy() 的函数来进行复制。对于 uvmdealloc() 来说，要注意用户页表负责释放底层物理页面，用户内核页表不能做重复的事。

    (注：内核部分映射和用户部分映射可以合二为一的重要前置条件是两者没有重复的映射。根据 xv6 手册，低于 0x80000000 的很多部分的内存是作为外设地址的。因此用户实际可用的内存空间其实远小于 MAXVA。式样讲义规定用户只能使用 PLIC 以下的地址。但事实上 CLINT 映射的地址比 PLIC 还要低，实验证明如果不取消 CLINT 的映射，用户内存会覆盖到这部分，触发 'remap' kernel panic。因此笔者只能在用户内核页表的映射中将 CLINT 移除。)

> What permissions do the PTEs for user addresses need in a process's kernel page table?  (A page    with `PTE_U` set cannot be accessed in kernel mode.)     

用户内核页表的页表项不能照抄用户页表，因为 QEMU 模拟的 RISC-V 硬件在内核态下是不能访问带有 PTE_U 位的页面的。因此用户内核页表的底层页表项都要将 PTE_U 挖去。