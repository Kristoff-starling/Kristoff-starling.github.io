---
layout: post
date: 2022-04-15
title: "MIT-6.S081 Lecture 11: Thread Switching"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Introduction

People want threads for the following reasons:

* People want their computers to "seemingly" simultaneously do multiple jobs.
* Programmers can use multiple threads to write simple and elegant code. `prime.c` in Lab 1 is a good example, it uses multiple processes but the principle is similar.
* We can utilize multiple threads to speed up our programs on a multi-core machine.

<!-- more -->

A thread is an abstract concept that can be viewed as **a serial execution**. The state of an execution includes the PC, the registers and a stack. Operating systems interleave the execution of multiple threads, which is implemented in mainly two ways:

* Multi-core machine: a machine with multiple CPU cores can naturally interleave the executions. There are automatically multiple sets of PCs, registers, etc. 
* Thread switching. Even on a multi-core machine, we need to focus on how a single core an execute multiple threads by executing the first one for a while and switching to the second and so forth.

An important issue about threads is: does threads have shared memory? If threads have shared memory, then each thread's changes are visible to other threads and we need locks to access shared resources. In xv6, kernel threads do have shared memories while user processes with only one thread don't have shared memory - each thread lives in the address space of its corresponding process. In real OS like Linux, which supports user processes to have multiple threads, threads of the same process share the same address space.

There are several challenges for implementing a threading system:

* How to decide which thread to execute - scheduling.
* What and where to save/restore the thread state.
* How to deal with compute bound threads: threads may not voluntarily yield the CPU ,so we need a method to automatically revoking control from some long running compute bound process, setting it aside and running it later.

The answer to the third question is **timer interrupts**. There is a piece of hardware on the CPU that periodically generates interrupt signals and the signals are somehow transferred to the kernel. So even if the process is running in user level, the coming signal will interrupt it and give the control to the kernel. Then the kernel voluntarily yields the CPU.

This kind of policy is called **preemptive scheduling**. Here the "preemptive" means that even if the user process is unwilling to yield the CPU, our kernel will force it to quit. Correspondingly another policy is called voluntary scheduling, which counts on processes to proactively yield the CPU. An interesting thing is, xv6 implements preemptive scheduling by letting kernel threads voluntarily yield CPU.

Kernel threads have multiple states, some of which are listed below:

* RUNNING: the thread is currently running on one of the core.
* RUNNABLE: the thread is able to run and wants to be scheduled onto CPU as soon as possible.
* SLEEPING: the thread cannot run currently (maybe because it's waiting for I/O).

Preemptive scheduling is responsible for changing a running thread to a runnable thread. In xv6, since kernel threads share memory, thread state only includes CPU information. A running thread's state is stored in the CPU hardware while a runnable/sleeping thread's state, i.e. PC and registers, is stored elsewhere.

## A Roadmap for Thread Switching

Generally, in xv6, a user process cannot directly switch to another user process, it must realize it in an indirect way: the user process first falls into the kernel, during which the user level information is stored in the trapframe, then it saves the **context** of its kernel thread, switches to another process's kernel thread by restoring the context. At last it goes back to the user space of another process.

A more detailed picture is described below: suppose we have two processes P1 and P2, P1 is running using its own user stack. Now a timer interrupt comes and the execution of P1 is suspended. In `trampoline.S`, it stores the user level information in the trapframe, switches to the corresponding kernel stack, and now it's P1's kernel thread running.

The kernel thread voluntarily yields the CPU by calling a function swtch(). In swtch(), the context of P1's kernel thread (i.e. values of registers) is saved and a **scheduler thread**'s context is restored. After restoring scheduler thread's context, our execution flow jumps to the return address of an swtch() call in scheduler thread's function scheduler(). Also, the stack is switched to the scheduler's stack (this stack is allocated in `entry.S`).

The scheduler thread is responsible for choosing an runnable thread and switching to it. After deciding which thread to switch to, the scheduler thread does the same procedure again: it calls swtch(), saves the context and restore the context of the target process's kernel thread. Now the execution flow jumps to the return address of an swtch() call in P2's kernel thread. Also the stack is switched to P2's kernel stack. Finally, P2 restores user level information through P2's trapframe and goes back to the user space.

> **Where are the context stored?**
>
> Contexts of processes's kernel threads are stored in p->context. Context of scheduler thread is stored in the CPU struct.

A process is either running in the user space, or running in the kernel level, or it's not running at all. In the last case the information of the process is stored in the combination of trapframe and thread context.

## Code: `spin.c`

```c
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

process struct and CPU struct are shown above. In `struct proc` we can see variable `state` representing the status of the process, `kstack` representing the address of the process's kernel stack, `trapframe` storing user level information and `context` storing kernel thread context. In `struct cpu` there's also a `context` storing the context of scheduler thread.

We use a demo program, `spin.c`, to illustrate the complete procedure of thread switching.

```c
//
// spin.c
// a demo program illustrating the principle of thread switching.
//

#include "kernel/types.h"
#include "user/user.h"

int
main (int argc, char *argv[])
{
    int pid;
    char c;

    pid = fork();
    if (pid == 0) {
        c = '/';
    }
    else {
        printf("parent pid is %d, child is %d\n", getpid(), pid);
        c = '\\';
    }

    for (int i = 0; ; i++)
        if (i % 1000000 == 0)
            write(2, &c, 1);
    
    exit(0);
}
```

In `spin.c` two processes print `/` and `\` respectively. Due to timer interrupt, two kinds of slashes are printed alternatively. We use GDB to set a breakpoint at where devintr() recognizes an timer interrupt.

> **Tip: GDB Usage**
>
> We can set breakpoints at a particular line in the source code. In this case, we can use `b trap.c:207`.

If we check the trapframe by `p/x p->trapframe`: 

```
$6 = {kernel_satp = 0x8000000000087fff, kernel_sp = 0x3fffff8000,  
  kernel_trap = 0x80002880, epc = 0x62, kernel_hartid = 0x0,        
  ra = 0x62, sp = 0x2fb0, gp = 0x505050505050505,                   
  tp = 0x505050505050505, t0 = 0x505050505050505,                   
  t1 = 0x505050505050505, t2 = 0x505050505050505, s0 = 0x2fe0,      
  s1 = 0x1c4a4111, a0 = 0x1, a1 = 0x2fbf, a2 = 0x1, a3 = 0x3ea0,    
  a4 = 0x1400, a5 = 0x99691, a6 = 0x505050505050505, a7 = 0x10,     
  s2 = 0xf4240, s3 = 0x20, s4 = 0x146b, s5 = 0x13f0,                
  s6 = 0x505050505050505, s7 = 0x505050505050505,                   
  s8 = 0x505050505050505, s9 = 0x505050505050505,                   
  s10 = 0x505050505050505, s11 = 0x505050505050505,                 
  t3 = 0x505050505050505, t4 = 0x505050505050505,                   
  t5 = 0x505050505050505, t6 = 0x505050505050505}
```

The epc points at 0x62, by looking up in `spin.asm`, this is `addiw s1, s1, 1` instruction, which demonstrates that our process is interrupted when doing infinite calculation.

We continue and go into the yield() function, which makes the kernel thread voluntarily yield the CPU. 

```c
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

Before doing jobs, yield() firstly acquires the lock of the current process because it's about to change the state of the process to RUNNABLE. We want this procedure to be atomic to other cores because the invariant that `p->state` represents the current status of the process is temporarily violated: the process is indeed running, but the status becomes RUNNABLE. If other cores' scheduler threads were able to see the change now and decide to execute this process, our current process would be running simultaneously on multiple CPUs, which is a disaster.

We go into sched(). After finishing several sanity checks, sched() calls swtch(), which is the core function of thread switching.

```c
swtch(&p->context, &c->context);
```

This line will save the current context into p->context and restore c->context. We can use `p/x cpus[0].context` to check the context we are going to switch to:

```
$12 = {ra = 0x80001fce, sp = 0x8000a7c0, s0 = 0x8000a810,
  s1 = 0x800121a0, s2 = 0x2, s3 = 0x80017768, s4 = 0x80011950,
  s5 = 0x1, s6 = 0x80011970, s7 = 0x1, s8 = 0x3, s9 = 0x0,
  s10 = 0x0, s11 = 0x0}
```

We are particularly interested in the `ra` register because it determines where we'll go to after returning from swtch(). We use `x/i cpus[0].context->ra`:

```c
0x80001fce <scheduler+138>:  sd      zero,24(s4)
```

We are assured that after executing swtch(), we will be in the scheduler thread.

`swtch.S` is written in assembly. It stores and restores values according to a0 and a1, which are registers containing the first and the second argument, i.e. p->context and c->context.

> **Why don't we save the PC register? Without saving the PC register, how can we know which instruction to jump to?**
>
> We use the `ra` register to indirectly record PC. When swtch() returns by executing `ret`, the value in `ra` will be put into PC, which is exactly the next instruction of `call swtch` in scheduler().

> **There are 32 general registers in RISC-V, why swtch() only saves 14 of them?**
>
> In C code, swtch() is only a normal function call, according to the calling convention, swtch() only need to carefully manage callee-saved registers. Even if swtch() do nothing with caller-saved registers, the caller will be able to restore the caller-saved registers (from the kernel stack or somewhere). This is ensured by the C compiler.

After executing 28 `ld` and `sd` instructions, we now use `x $sp` to print the stack:

```
0x8000a7c0 <stack0+3984>:       0x00000000
```

our stack pointer now points to stack0, which is the stack we allocate to the scheduler thread during booting. If we use `where` to unwind the stack information, we'll get

```
#0  0x00000000800026b8 in swtch ()
#1  0x0000000080001fce in scheduler () at kernel/proc.c:489
#2  0x0000000080000f2c in main () at kernel/main.c:44
```

We're now in scheduler thread, which is created at the last line of `main.c` during booting. We return from a swtch() call which scheduler() makes a while ago.

scheduler() loops and tries to find process that is RUNNABLE state. It's worth attention that we need to acquire lock before checking the status and doing subsequent jobs. Generally, after we acquire a process lock, we usually do the following jobs to realize thread switching:

* Modify the status of the target process.
* Call swtch() to save and restore context.

Previously we talked about acquiring lock can avoid other cores discovering processes in temporary incorrect state. Another significant point of acquiring lock is that we explicitly turn off interrupts in acquire(), this is important because we don't want to respond to timer interrupts when doing swtch(), otherwise the register values would be damaged.

After scheduler() decides the target process to switch to, it will call swtch() again, save scheduler thread's context into c->context, restore context in p->context (another process's kernel thread's context), and resume executing that process.

> **How can a process be switched to at the first time, since it hasn't called swtch() yet?**
>
> In xv6 there is a function forkret(), it serves as the target address of the first "fake" context: in allocproc(), we set up a fake context with `ra` pointing to forkret() and `sp` pointing to the corresponding kernel stack. We don't care about other registers' values before the first execution. The only thing forkret() does is to release the process lock. 
>
> In addition, there is a situation in which xv6 creates a "fake" trapframe: in userinit(), xv6 creates a fake trapframe with `epc=0` and `sp=PGSIZE`. In other situations, a new process is created by fork() and fork() copies the trapframe of parent process.

