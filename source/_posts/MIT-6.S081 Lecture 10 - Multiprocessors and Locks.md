---
layout: post
date: 2022-03-12
title: "MIT-6.S081 Lecture 10: Multiprocessors and Locks"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

Applications may want multiple CPU cores, so the kernel must handle parallel system calls, e.g. carefully maintain shared data structure in parallel. We use locks to ensure correct sharing, but on the other hand, locks can limit performance.

The frequency of CPU clock gradually reaches a limit at 2010, which means that single thread performance cannot be optimized. However, the number of transistors keeps growing, so having multiple cores is our only choice to increase performance.

<!-- more -->

## Why Lock?

We use lock to avoid race conditions. For example, if we delete the `acquire(kmem.lock)` and `release(kmem.lock)` in kfree(), it's possible that pages are lost during freeing.

## Lock Abstraction

The lock in xv6 is simply a struct:

```c
struct lock
{
    int locked;
    ...
};
```

there are two APIs: `acquire(struct lock *lk)` and `release(struct lock *lk)`，at one time, only one process can acquire the lock, and other processes have to wait until the first process calls release().

The area between acquire() and release() is called critical section. The code in critical sections are executed **atomically**, i.e. either **all or none** of the code is executed, it's impossible for codes in the same critical section to interweave.

Programs usually have many locks, aiming at more parallelism. If there's only one lock in the kernel, to ensure correctness, the lock must protects all the codes in the kernel (the so called 'big kernel lock'). In this case, the execution of the whole kernel is serialized. If we use multiple locks, it's allowable to let two processes using different locks run in parallel.

## When to Lock?

A conservative rule (guideline): if 2 processes access a shared data structure and at least one operation is writing, then we have to lock that data structure.

From some aspects, the guideline is too strict: we can use lock-free programming. But lock-free programming is so tricky that we won't cover that. From some aspects, the guideline is too loose: in printf() we use lock to ensure that the printed message is atomic.

We may want data structures to automatically acquire and release locks when we're accessing them, but there are cases when we need more flexible lock management. Consider a system call `rename()` . Suppose now we deal with `rename("d1/x", "d2/y")`, if we act like this:

```
lock(d1); erase("d1/x"); release(d1); (**)
lock(d2); add("d2/y"); release(d2);
```

then at time `**`, from other processes' perspective, the file doesn't exist! - obviously incorrect. The correct implementation is

```
lock(d1 and d2); erase("d1/x"); add("d2/y"); release(d1 and d2);
```

In this example, we see that sometimes we need multiple locks before doing operations.

## Lock Perspectives

* Locks help avoid lost updates. (e.g. kfree() )
* Locks make multi-step operations (critical section) atomic.
* Locks help maintain invariants. (e.g. in kfree(), the lock protects moments when there are multiple pointers pointing to the head node.)

## Deadlock

Let's look at a case which may lead to deadlock:

```
CPU1                            CPU2
rename("d1/x", "d2/y")          rename("d2/a", "d1/b")
acquire(d1)                     acquire(d2)
acquire(d2) <-- blocked         acquire(d1) <-- blocked
```

The solution is to always acquire locks in order. (actually, it's OK for CPU2 to acquire d1 and then d2 in this case.) Here the lock ordering is a partial order.

Lock makes modularity more complicated. Suppose a function f() in module 1 calls a function g() in module 2, f() must ensure that locks in g() will not violate global lock ordering, which means that the internals of module 2 (in terms of locks) must be visible to module 1, which somehow violates the module abstraction principle.

## Lock Performance

Locks lead to serialization, which influences performance. To increase performance, we need to split up data structures and introduce more locks, which require lots of work.

A general approach is to firstly use coarse-grained locks, and do profiling tests to see whether we need smaller locks to increase parallelism.

## Code: Lock Implementation

The most common way to acquire locks is to use the hardware test-and-set support. In RISC-V, we use the `amoswap addr, r1, r2` atomic instruction. It can be viewed as the following pseudo instructions:

```
lock addr
	tmp   <-- *addr
	*addr <-- r1
	r2    <-- tmp
unlock addr
```

Hardware test-and-set support is dependent on memory system. If the memories are sitting on a share bus, a memory controller (bus arbiter) is responsible for sorting the locks in order. If processors have caches, the cache coherence protocol will ensure the consistency.

In acquire() in xv6, we use the C function `__sync_lock_test_and_set()` to do atomic operations. Refer to Xv6 Source Code Manual for more details.  

In release(), we may wonder that why we still need to use atomic instruction to set the lock: a simple store instruction can finish that. However, the store instruction may be executed differently depending on architecture design. `sw` may be decomposed to several micro instructions, which leads to lost of atomicity.

## Code: Locks and Interrupts

In acquire(), we must firstly turn off the interrupts. That's because unexpected interrupts may cause deadlock. For example, in `/kernel/uart.c`:

```c
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }
  ......
}

void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}
```

Suppose that we don't turn off interrupts when holding locks, we acquire the lock `uart_tx_lock` in uartputc(), and when we're in the critical section, the uart hardware sends an interrupt, leading us to uartintr(). In uartintr() the kernel tries to acquire the lock `uart_tx_lock`, but the lock is held by the interrupted thread, on the other hand, it's impossible for the interrupted thread to move on unless interrupt handler returns. So a deadlock appears.

## Code: Memory Ordering

Modern compilers and processors usually adjust instruction order to increase performance. For example,

```c
lock = 1;
x = x + 1; // ---+
lock = 0;  //    |
           // <--+		
```

It's possible for compilers/processors to move `x = x + 1` to the end of the paragraph. If the code is executed in a single-core, serial mode, it's absolutely OK and correct. But if it's executed on mult-core processors, it's a disaster.

To avoid such things, we can build a memory fence. In RISC-V, it's a `mfence` instruction and it's encapsulated as `__sync_synchronize()` in libc. We put memory fence at the beginning of acquire() and the end of release() to ensure that no instruction will be unexpectedly moved out of our critical section.