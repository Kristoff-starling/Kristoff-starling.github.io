---
layout: post
date: 2022-01-26
title: "MIT-6.S081 Lecture 03: OS Organization and System Calls"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Isolation

* We want strong isolation between applications so that if there's a bug in one application, the others will not be damaged.
* We also want isolation between applications and the operating system so that if weird arguments were passed to the OS, OS wouldn't crash.

<!-- more-->

Let's consider what happens if there's no operating system, a strawman design. In this case, several applications (e.g. `shell`, `echo` etc.) run on the hardware, there is no abstraction layer between H/W and applications. 

* There's a demand for switching between applications. Because there's no OS, `shell` needs to proactively "give up" and "quit" the CPU to let others use the hardware. This is the so-called cooperating schedule. However, if there's a dead loop in `shell` or `shell` is a malware, it will not quit and the hardware is completely occupied by this application. Therefore, we need a kind of "enforced multipliexing": **no matter what the application does, it will be forced to give up the CPU once in a while**.

* Different applications share the same memory space. For example, `shell`'s memory may start at address 1000 and `echo`'s memory may start at address 2000. There's no boundary between their memory space and it's easy for one application to overwrite others' memory and cause tricky bugs.

In summary, we need operating systems to **enforce both multiplexing and strong memory isolation**. Strawman design is not very common in today's computer system. Some real-time system doesn't have OS because applications trust each other, but in most cases we need OS to ensure isolation.

Unix inferfaces are sophisticated design. They abstracts the hardware resources and make isolation possible. For example,

* Process is the abstraction of CPU. Applications use `fork()` to create new processes, but it cannot assign a CPU core directly. How to schedule the processes on the CPU and how to arrange different CPU cores are the jobs of OS.
* `Exec` is the abstraction of memory. Application use `exec()` to load a new applications, during which it loads the **memory image** which describes the text, data segments etc. During execution, applications can use system calls like `sbrk()` to modify the memory image, but they cannot act directly on the physical memory. OS is responsible for allocating physical memory to the memory image.
* File is the abstraction of disk blocks. Applications do operations on the files but they don't have direct access to the physical storage device. They don't know where the "file" is actually stored, which is the job belongs to OS.

OS should be defensive: it should treat all the processes as malicious applications written by attackers and ensure that

* Apps cannot crash the OS.
* Apps cannot break out of the isolation.

Strong isolation is totally depend on the OS. The kernel is call "trusted computing base" (TCB). Kernels must have as few bugs as possible because any bug may be taken advantage of and becomes an exploit.

Strong isolation between applications and the OS are typically implemented by hardware support. Ususally there're two: use/kernel mode and virtual memory.

### User/Kernel mode

When running in kernel mode, CPU can execute privileged instructions which interact directly with CSRs e.g. setting up page table register, disable clock interrupts etc. 

In user mode, CPU can only execute unprivileged instructions such as `add` `jmp` etc. There is a bit in the CSR to let hardware check which mode the current process/thread is in (0 for kernel, 1 for user). If the CPU decode one instruction, find out that it's a privileged one and the mode bit is 1, it will deny executing the instruction. (Of course, the instruction for changing the mode bit is a privileged one.)

If a user application wants to execute privileged instructions, it should transfer from user mode to kernel mode first and let the kernel execute the instruction. 

There is a method for applications to enter the kernel. In RISC-V, the instruction is `ecall`. The user store ssystem call number in a particular register and `ecall`, the PC jumps to a specific address in the kernel, and kernel checks the syetem call number, does security checks and jumps to corresponding service functions.

### Virtual Memory

Each process has its own page table: it maps virtual addresses to physical addresses. If the OS allocates different memory space to different processes, processes will not be able to access other processes's memory space because those addresses are not in its page table. In this way, the OS provides strong memory isolation.

## Kernel Design

An instant question is: since kernel should be TCB, what kind of code should be run in kernel mode? Most Unix-like operating system put the whole OS into the kernel mode(including xv6). This style is call monolithic kernel design. Though this design contains more code and increases the risk of having serious bugs, it's helpful for tight integration between different modules (file system, virtual memory, processes etc.) and can achieve better performance.

Another style, which aims at running as fewer lines as possible in the kernel to avoid bug risks, is called micro kernel design. The kernel only contains some core modules and other modules like file system are run in user mode and treated like ordinary user applications. The challenge for this design is how to improve performance. For example, if an application wants to access the file system, frequent jumps between kernel mode and user mode are needed to achieve that. (app(u) $\rightarrow$ IPC(k) $\rightarrow$ FS(u) $\rightarrow$ IPC(k) $\rightarrow$ app(u))

## Xv6 Code Details

Xv6 uses monolithic kernel design. All the `*.c` programs in `/kernel` are run in kernel mode. The Makefile grabs all the C files in `/kernel` and generates relocatable files: `*.c`$\overset{CC}{\rightarrow}$`*.S`$\overset{AS}{\rightarrow}$`*.o`. ld is responsible for linking all the `*.o` files and generate an executable file `/kernel/kernel`. `/kernel/kernel.asm` containing disassembled code of the kernel will also be generated for debugging.

Xv6 starts at address 0x80000000. The code is written directly in assembly and is in `/kernel/entry.S`. `/kernel/kernel.ld` ensures that the assembly code will be linked to 0x80000000. Currently, xv6 runs in machine mode - no protection, no virtual memory etc. The job for xv6 is to enter supervisor mode(kernel mode) as soon as possible.

When xv6 jumps to `main()` in `/kernel/main.c`, it has been in supervisor mode. Here we use `make CPUS=1 qemu-gdb` to fire up xv6, so there's only one CPU core in our simulated CPU. The code in `main()` is shown as below:

```c
if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode cache
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
}
```

Xv6 initializes lots of things and then jumps to `userinit()` run the first process. `userinit()` (in `/kernel/proc.c`) loads the first process. Because currently xv6 hasn't built the file system and cannot load images, the assembly code is statically declared in the array `initcode[]`. The assembly code of the "first process loader" is in `/user/initcode.S`:

```assembly
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```

`initcode.S` prepares arguments for the `SYS_exec` system call and uses `ecall` instruction to go back to kernel space. `ecall` will lead us to `syscall()` in `/kernel/syscall.c`:

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

The variable num's value, after the instruction `num = p->trapframe->a7`, equals 7, which is `SYS_exec`'s value defined in `/kernel/syscall.h`. The kernel knows that some user program wants to use the exec system call, so it goes to `syscalls[num]()`, which is a function table for providing system call services.

The function `sys_exec()` is in `/kernel/sysfile.c`. In line 444:

```c
int ret = exec(path, argv)
```

If we print the value of "path", we will find that it equals "/init". So xv6 loads user program `/user/init.c`.

`init.c` mainly forks some child processes and exec `sh.c`, so xv6 starts the shell and the booting procedure is completed.

