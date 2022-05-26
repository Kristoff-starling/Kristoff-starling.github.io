---
layout: post
date: 2022-02-01
title: "MIT-6.S081 Lecture 07: Q&A for Labs #1"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

The Q&A focuses on lab pgtbl. The lab contain few lines of code, but hard-to-debug problems. Harsh environment of kernel programming makes it more difficult.

<!-- more -->

## Task 01

```
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

* 0-0-0: text and data of the `init` program.
* 0-0-1: the guard page (of the user stack)
* 0-0-2: user stack (tighter privilege can be set: PTE_X flags can be erased, the user stack is not supposed to store instructions.)
* 255-511-510: trap frame
* 255-511-511: trampoline

One interesting fact is that: the physical pages allocated to the continuous virtual pages are not continuous. That's the magic of virtual memory.

> **Q: The trampoline page is at the highest virtual memory address MAXVA, why the first page directory's index is 255 instead of 511?**
>
> RISC-V Sv39 support 39 bits virtual address, but xv6 only uses 38 bits to avoid dealing with sign extension. Therefore, the first page directory's index is only up to 255.

> **Q: Why the text page and data page are merged into one page in `init`?**
>
> Generally, separate text page and data page can support more careful privilege settings. But in `init` program, we don't use the `exec()` system call to load memory image, for simplicity of code in userinit() function, we combine text and data into one page.

## Task 02

One possible approach is to just copy the mappings in the kernel page tables that keep unchanged.

```c
pagetable_t
kvmcreate()
{
	pagetable_t pagetable;
	int i;
 	pagetable = uvmcreate();
    for (i = 1; i < 512; i ++) pagetable[i] = kernel_pagetable[i];
    
    kvmmapkern(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
    kvmmapkern(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
    kvmmapkern(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
    kvmmapkern(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
    
    return pagetable;
}
```

For the creation of a user kernel page table, we're sure that later we only use the range `[0, PLIC)` for user memory, so only the first page directory entry may change. We can directly copy the kernel's first level page directory of range `[1,512)` . Don't forget to map several I/O devices in the first page directory entry.

```c
void
kvmfree(pagetable_t kpagetable, uint64 sz)
{
    pte_t pte = kpagetable[0];
    pagetable_t level1 = (pagetable_t) PTE2PA(pte);
    for (int i = 0; i < 512; i ++)
    {
        pte_t pte = level1[i];
        if (pte & PTE_V)
        {
            uint64 level2 = PTE2PA(pte);
            kfree((void *)level2);
            level1[i] = 0;
        }
    }
    kfree((void *) level1);
    kfree((void *) kpagetable);
}
```

To free the user page table, we don't need to do anything about the 1~511 page directory entries since they're just copied from the kernel page table. We need to walk through the first page directory entry to free the remaining pages. (Notice: kvmfree() is only responsible for freeing page table pages, the leaf physical pages for user memory should be beforehand freed.)

In the function scheduler(), we should switch to user kernel page table before swtch() and switch to kernel page table after swtch():

```c
kvmswitch(p->kpagetable);
swtch(&c->context, &p->context);
kvminithart();
```

> **Q: Why must we switch back to the kernel page table after going back?**
>
> If there's no running process, the kernel should have an page table to use. Also, if we want to free a process, we must firstly switch back to the kernel page table before the process "disappears".

## Task 03

The advantages of adding user mappings into user kernel page table include:

* during copyin()/copyout(), we don't need to call walk() in the kernel to translate the user virtual address since now hardware can do the same thing for us, which is a relief to kernel programmers.
* performance is enhanced since we do less page table walks.
* kernel has more freedom to manipulate with user space. it can directly read/write data without repeatedly calling copyin()/copyout().

The core function should be responsible for copying mappings from user page table to user kernel page table:

```c
void
kvmmapuser(int pid, pagetable_t kpagetable, pagetable_t upagetable, uint64 newsz, uint64 oldsz)
{
    uint64 va;
    pte_t *upte;
    pte_t *kpte;
    
    if (newsz >= PLIC)
        panic("kvmmapuser: news too large");
   	
    for (va = oldsz; va < newsz; va += PGSIZE)
    {
        upte = walk(upagetabe, va, 0);
        if (upte == 0)
            panic("kvmmapuser: no upte");
       	if ((*upte & PTE_V) == 0)
            panic("kvmmapuser: no valid upte");
       	kpte = walk(kpagetable, va, 1);
        if (kpte == 0)
            panic("kvmmapuser: no kpte");
        *kpte = *upte;
        *kpte &= ~(PTE_U|PTE_W|PTE_X);
    }  
}
```

The line `*kpte = *upte` means that user physical pages are shared between user space and kernel space although the kernel allocates new pages for page directories/tables.

The PTE_U bit is cleared because in RISC-V hardware, kernel doesn't have the privilege to read/write pages with PTE_U flag. The reason for this mechanism is not isolation, but convenience for kernel debugging. Xv6 without modification should never write/execute user memory. 

> **Q: Currently we require that the user memory space cannot exceed PLIC. What if we want to use all the space below KERNBASE?**
>
> We can remap the I/O devices: CLINT, PLIC, UART0 etc. Actually there is still much space between PHYSTOP and the kernel stacks. Put I/O devices there in the kernel's virtual address space and map them to low physical addresses. In this case, low virtual addresses are available for user memory.

> **Q: Why should we put the kernel stacks high in the virtual address space?**
>
> Below every kernel stack there is a guard page, which is not mapped to any physical page, in order of detecting stack overflow issue. If the guard pages are put in the range of `[KERNBASE, PHYSTOP)`, physical pages will be wasted.