---
layout: post
date: 2022-01-30
title: "MIT-6.S081 Lecture 06: Traps"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Introduction

When going into traps, we need to carefully change the hardware state so that isolation will not be broken. The hardware state of RISC-V machine includes:

* 32 general registers

    We keep a copy of general registers because we want to resume the user code later transparently, particularly when it's a unexpected device interrupt.

* PC

* Mode: we need to switch to supervisor mode.

    Supervisor mode has the following privileges:

    * read/write control registers
    * use PTEs without PTE_U sign.

* Page table (satp): page table currently points to the user page table, which only contains user's code and data. To run kernel code, we need to switch to kernel page table.

* stvec

* sepc

* sscratch

<!-- more -->

There are some high-level principles of trap design:

* **We cannot count on anything in the user space**. We can't assume anything about the registers since they are likely to contain malicious values. In xv6, the trap handler just saves them.
* We want to be **transparent** to the user code.

## The high-level procedure of a system call

Shell $\rightarrow$ write() $\overset{ecall}{\rightarrow}$ uservec (asm) $\rightarrow$ usertrap() $\rightarrow$ syscall() $\rightarrow$ sys_write() $\rightarrow$ syscall() $\rightarrow$ usertrapret() $\rightarrow$ userret (asm) $\rightarrow$ the next instruction of `ecall`.

> **System call is quite expensive. So can we return some address mappings instead of a file descriptor in `open()` , so that the user program can directly write to the memory without using system calls?**
>
> Yes. Modern operating systems support this mechanism and it's called memory mapped file access. It makes read/write much faster compared with the file descriptor method.

## Code: `write()` system call

### Preliminaries

The shell program uses `write()` system call to print a `$` to the console.

```c
static void
putc(int fd, char c)
{
  write(fd, &c, 1);
}
```

write() function is defined in `usys.asm`

```assembly
write:
 li a7, SYS_write
 ecall
 ret
```

It loads the system call number "SYS_write" into register `a7`. `a0` `a1` `a2` stores the arguments of the system call. In GDB, we can use `x/2c $a1` to print the argument strings.

> **GDB Tips**
>
> Use `b *addr` to set a breakpoint at a specific address. This method is useful when we want to set breakpoints at particular instructions.

Before switching to kernel, we are using the user page table. We cannot print the page table in GDB, but luckily, QEMU provides functions to print the current page table. Type `<c-a>` `c` to get into the QEMU monitor, and type `info mem` to print the page table:

```
vaddr            paddr            size             attr
---------------- ---------------- ---------------- -------
0000000000000000 0000000087f60000 0000000000001000 rwxu-a-
0000000000001000 0000000087f5d000 0000000000001000 rwxu-a-
0000000000002000 0000000087f5c000 0000000000001000 rwx----
0000000000003000 0000000087f5b000 0000000000001000 rwxu-ad
0000003fffffe000 0000000087f6f000 0000000000001000 rw---ad
0000003ffffff000 0000000080007000 0000000000001000 r-x--a-
```

The last column shows the privilege flags of PTEs. The first and second PTEs are code and data page of the shell program. The third page, which doesn't have the PTE_U flag, is the guard page below the stack page. The fourth page is the stack page. The last two pages, which have high virtual addresses, are the trap frame and the trampoline page. Those pages also don't have PTE_U flag and can only be accessed by the kernel.

> **PTE's PTE_A & PTE_D flags**
>
> xv6 doesn't use these flags. These flags are used for page exchanges when there's a page fault and the physical memory pages are used out.

### Trampoline

We're now ready for executing `ecall`. After executing `ecall`, our PC register goes to the trampoline page: 0x3fffff000. `ecall` doesn't switch the page table, which means that the trampoline page must be mapped in the user page table. Although user page table contains this mapping, the corresponding PTE doesn't have PTE_U flag so this page is protected against user code.

`ecall` actually does 3 things:

* Switch from user mode to supervisor mode.
* Save the value of PC into sepc.
* Jump to the address held in stvec register.

`ecall` only does the minimum things. Some hardware does more when triggering a system call: they save the registers, switch the page table and set the stack pointer. RISC-V doesn't choose to include those parts into the hardware because it wants the software to achieve the maximum flexibility. For example:

* In some system calls the kernel can finish jobs without the help of kernel page table. Also, some operating systems combine the user page table and kernel page table together, which means they don't need to change the page table during system calls.
* According to the specific function, some system calls won't use all the general registers and some registers' values don't need to be saved.
* Some easy system call may not require a kernel stack. Stack switching is not necessary.

Although xv6 doesn't utilize those freedom, modern operating systems care a lot about the performance. There are a lot of sophisticated, clever schemes that make full use of the flexibility and achieve better efficiency.

If we check the general registers' values, we'll find that they are the same as those in the user space. Currently we cannot modify any register, otherwise the user data would be damaged. We should firstly store them somewhere and restore them before returning. 

Some hardware may switch to the kernel page table and store the 32 values somewhere in a physical page. But now we even don't know the address of the kernel page table! xv6's solution for saving registers includes two parts:

* Use a trap frame. User page table contains the mapping of this per-process trap frame. The trap frame contains 32 slots for register values, and some other useful values such as the address of kernel page table.
* The (virtual) address of the trap frame is beforehand stored in sscratch register. xv6 utilize `csrrw` instruction to atomically swap the values in a0 and sscratch. So sscratch temporarily stores the old value of a0 and a0 contains the address of the trap frame. We now can use `sd` instruction to save register values in the slots.

> **Q: Where does the kernel set the value of sscratch register?**
>
> The function usertrapret() in `/kernel/trap.c` sets control registers including stvec, sscratch etc.
>
> ```c
> w_stvec(TRAMPOLINE + (uservec - trampoline));
> 
> p->trapframe->kernel_satp = r_satp();         // kernel page table
> p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
> p->trapframe->kernel_trap = (uint64)usertrap;
> p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()
> 
> unsigned long x = r_sstatus();
> x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
> x |= SSTATUS_SPIE; // enable interrupts in user mode
> w_sstatus(x);
> 
> w_sepc(p->trapframe->epc);
> 
> uint64 satp = MAKE_SATP(p->pagetable);
> uint64 fn = TRAMPOLINE + (userret - trampoline);
> ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
> ```
>
> The last line calls userret() and transmits two arguments including the address of TRAPFRAME, in userrret(), TRAPFRAME is stored into sscratch register.

> **Q: Every time after `ecall`, we have a chance to execute usertrapret() and set the correct value in sscratch, but how do we set it correctly before the first execution?**
>
> When xv6 is booting, it firstly goes into the supervisor mode. So the kernel also use usertrapret() to start executing a user program. usertrapret() is the only way xv6 provides to enter into the user program from the kernel.

> **Q: Why should we use a particular trap frame instead of saving the values on the user stack?**
>
> Not all the program languages have the same stack structure. Some program languages don't have a stack, and some languages organize stack space in a weird way that kernel cannot understand (e.g. by allocating memory blocks). The principle for trap design is that **the kernel cannot count on anything from the user space**. The safest way for kernel is to save the values in the trap frame - a frame that belongs to the kernel and is reliable.

(Author's question: can we use a control register in RISC-V to store the address of kernel stack and save the register values on the kernel stack? In this way we don't need an additional trap frame?)

After saving registers, the kernel use `ld sp, 8(a0)` to load the address of kernel stack into the sp register. If we type `print/x $sp` in GDB, we'll see that `sp = 0x3fffffc000`. Recalling the structure of kernel space mappings, the address is reasonable - just below the trap frame, there's a guard page and the first kernel stack.

Next, the hart id is stored into tp register. The address of the function to be jumped to (usertrap()) is stored into t0 register. `csrw satp, t1` switches to the kernel page table.

> **Q: Why doesn't the kernel crash since we keep using user's virtual addresses?**
>
> That's because we're executing in the trampoline page. The kernel and user page tables both have mappings to the trampoline page. (The same virtual address) The page is named "trampoline" because xv6 bounces on this page between user code and kernel code.
>
> An important thing to remember is that: **page table switching can only be implemented in the trampoline** because the virtual address in this page maps to the same physical page in kernel and user page tables. Otherwise, the instruction addresses could not be interpreted and the kernel would crash.  

### usertrap()

After executing `jr t0`, we jump to the C-code function usertrap(). The first thing we do is

```c
w_stvec((uint64)kernelvec)
```

which overwrites the stvec register. Xv6 handles traps differently depending on whether they come from user space or the kernel.

```c
struct proc *p = myproc();
p->trapframe->epc = r_sepc();
```

These lines save the value of sepc into the trap frame. It's necessary because the kernel may switch to another process. After returning to the user space of that process, the user program may do another system call, and the sepc register will be overwritten. (Note: it's also correct to save sepc in the trampoline assembly code.)

Here we're doing `write()` system call, so the value in scause register is 8. We firstly add 4 to the sepc value in the trap frame in order not to re-execute the `ecall` instruction. Then we call intr_on() to accept interrupts. <u>Interrupts are always turned off by the RISC-V trap hardware, so we need to explicitly open it in the code</u>.

Xv6 enters syscall() in `/kernel/syscall.c` to handle system calls. In syscall(), xv6 retrieves the value in the a7 register (context in trap frame) to acquire the system call number and uses that to index the syscalls[] function array. After returning from sys_write(), the return value is stored in p->trapframe->a0.

### usertrapret()/userret()

After handling the system call, xv6 enters usertrapret(). The job for usertrapret() is to set control registers to appropriate states for user mode. At the end, it calls userret() in `/kernel/trampoline.S` to resume registers. Refer to code details in "Xv6 Source Code Manual".