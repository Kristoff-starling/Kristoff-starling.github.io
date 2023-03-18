---
title: "Lecture 05: Concurrent Programming: Mutex"
linktitle: '05: 并发-互斥'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 5
---

从 high level 来看，我们希望实现如下的结构体和 API：

```c
typdef struct
{
    ...
}lock;
void lock(lock *lk);
void unlock(lock *lk);
```

若 lock() 函数返回，说明获得了这个锁，别的进程会停在 lock() 里直到获得锁的人调用了 unlock()。

实现互斥的根本困难：不能同时读写共享内存。

* load 的时候不能 store，且看完之后眼睛立刻闭上，看到的都是过去的东西；
* store 的时候不能 load，贴字条的时候不知道覆盖了什么。

## Spin Lock

> “解决一个问题有两种方法：一个是分析问题、提出算法、实现解决，一个是解决提出问题的人。“ —— jyy

我们希望硬件支持一条“瞬间完成读和写”的指令：“请所有人闭眼，然后我睁开眼看一眼，并做一个操作"。

### x86: Lock Prefix

以 lock 为前缀的指令支持原子操作。

```c
// sum-atomic.c
#include "thread.h"

#define N 100000000

long sum = 0;

void Tsum() {
  for (int i = 0; i < N; i++) {
    asm volatile("lock addq $1, %0": "+m"(sum));
  }
}

int main() {
  create(Tsum);
  create(Tsum);
  join();
  printf("sum = %ld\n", sum);
}
```

由于这里的 +1 操作使用了原子指令，所以程序总能得到 2N 的输出。

此外，一条非常好用的原子指令是 xchg，它可以原子地将一个数值和一个内存地址上的值交换。用 C 语言封装后的形式可以是：

```c
int xchg(volatile int *addr, int newval) {
  int result;
  asm volatile ("lock xchg %0, %1"
    : "+m"(*addr), "=a"(result) : "1"(newval)); //xchg默认原子，即使不加前缀lock也可以
  return result;
}
```

借助 `xchg` 指令，我们可以轻松地实现一个自旋锁：

```c
int lock = 0;
void lock() {while (xchg(&lock, 1));}
void unlock() {xchg(&lock, 0);}
```

在硬件层面，硬件可以保证所有的带 lock 的指令可以排出一个全序，且原子指令相关的内存都会被提前做好 (内存的可见性)。但这其中有很多技术细节，比如多核的 Cache 之间的一致性是 x86 的噩梦 (一旦某个 CPU 要获得所，其他所有 CPU 核的 L1 Cache 都要上锁)。

### RISC-V: LR/SC

原子指令内部的本质是三个步骤：load(), exec(), store()。RISC-V 提供了 load-reserved/store-conditional (LR/SC) 机制：

* LR: 在共享内存上打一个标记，如果出现中断、别的线程写入等情况，这个标记会消失。
* exec()：在寄存器上做一些自己的事情
* SC：如果共享内存上的标记还在，则 store。

这里给出一个用于自旋锁的 compare-and-swap (旧值等于某个值才将新值与其交换) 函数的实现，用这个函数实现自旋锁在轮询时可以比 xchg 少一次写入操作：

```c
// C语言语义
int cas(int *addr, int cmp_val, int new_val)
{
    int old_val = *addr;
    if (old_val == cmp_val) {*addr = new_val;return 0;}
    else return 1;
}
```

```assembly
cas:
	lr.w t0, (a0)		# 获得旧值，并对(a0)打标记
	bne t0, a1, fail	# 如果 old_val != cmp_val 则跳转到 fail
	sc.w t0, a2, (a0)	# 如果标记还在，将a2写入(a0)，t0存储标记存在情况
	bnez t0, cas		# 如果标记被冲刷了，说明出现race condition，回到cas重做
	jr ra
fail
	li a0, 1
	jr ra
```

如果有两个线程同时希望获得锁，第一个线程执行完 sc.w 后，第二个线程的标记被冲刷，从而第二个线程执行 sc.w 后 t0=1，无法获得锁。

LR/SC 机制不仅可以实现锁，还可以让每个线程知道锁的拥堵情况。

## Sleep Lock

自旋锁的性能缺陷：

* 如果多个线程同时争抢锁，并行/并发的程序会因为自旋锁变得串行。除了进入临界区域的线程，其他的线程都在空转。
* 如果持有自旋锁的线程被调度走了，那么所有的线程都进不去了。

自旋锁的理想使用场景：

* 临界区几乎不拥堵。
* 持有自旋锁时禁止执行流切换 - 这件事对于应用程序来说太危险了，因此自旋锁一般用于操作系统内核中的并发数据结构 (临界区很短)。

---

如果锁被占用，待在原地等待锁被释放是一件不太聪明的事情 (尤其是要等很长时间的时候)，我们希望一个线程在等待锁的时候可以被暂时挂起，让出 CPU，等到锁被释放时再唤醒等锁进程。

C 语言代码是做不到 ”让出 CPU 的"，因此我们需要两个系统调用：

```c
syscall(SYSCALL_lock, &lk);		// 试图获得lk，如果失败则当前进程被挂起
syscall(SYSCALL_unlock, &lk);	// 释放lk，并唤醒正在等待lk的进程
```

操作系统的工作是维护这个锁和等待锁的进程

* 如果有线程尝试获得锁，且锁处于空闲状态，就把锁给他。
* 如果有线程尝试获得锁但锁已经被取走了，则将当前线程加入等待锁的队列，并将其挂起。
* 如果有线程释放了锁，就检查等待锁的队列中是否有线程，如果有则将锁交给下一位 (唤醒它)，如果没有则锁的状态为空闲。
* 操作系统使用自旋锁保证自己对队列的操作是原子的。

> **Tip: GDB Usage**
>
> 使用 GDB 调试多线程时，如果直接使用 next 命令，所有的线程都会向后走一步。为了更好地观测程序行为，我们可以使用 `set scheduler-lock on` 命令来打开线程锁，之后再使用 next 命令就只有当前线程会向前走一步。我们可以通过 `info threads` 来查看线程信息，通过 `thread id` 来切换当前调试的线程为 id。

## Futex

* 自旋锁有更快的 fast path：如果获得锁成功，可以立刻进入临界区；但如果没有获得锁，则要浪费 CPU 时钟周期。
* 睡眠锁有更快的 slow path: 如果没有获得锁，可以执行线程切换；但在能获得锁的情况下，也需要执行系统调用陷入内核。

我们希望发明一种锁，对于 fast path，只需要一条原子指令就可以获得锁，对于 slow path，可以通过系统调用 yield。这就是 Futex = spin + mutex。

我们可以利用 model checker 来确认 futex 的正确性：

```python
class Futex
	locked, waits = '', ''

    def tryacquire(self):
        if not self.locked:
            # Test-and-set (cmpxchg)
            # Same effect, but more efficient than xchg
            self.locked = '🔒'
            return ''
        else:
            return '🔒'

    def release(self):
        if self.waits:
            self.waits = self.waits[1:]		  # 将队首元素踢出队列 <=> 将编号为队首元素的线程唤醒
        else:
            self.locked = ''

    @thread
    def t1(self):
        while True:
            if self.tryacquire() == '🔒':     # User
                self.waits = self.waits + '1' # Kernel
                while '1' in self.waits:      # Kernel
                    pass
            cs = True                         # User
            del cs                            # User
            self.release()                    # Kernel

    @thread
    def t2(self):
        while True:
            if self.tryacquire() == '🔒':
                self.waits = self.waits + '2'
                while '2' in self.waits:
                    pass
            cs = True
            del cs
            self.release()
```

> **Tip: 强大的 model checker**
>
> * 我们的 model checker 的原理是：对于有装饰器 thread 修饰的函数，在每条语句后添加 yield。因此我们可以轻松地实现一个原子的操作：只要将操作写成一个函数，并在一行内调用这个函数即可。再例如，xchg 原子操作在我们的 model checker 中可以很简单地写为
>
>     ```python
>     x, y = y, x
>     ```
>
> * 线程函数中的语句
>
>     ```python
>     while '1' in self.waits:
>         pass
>     ```
>
>     并不是内核的真正行为：内核会切换走去执行别的线程。但从这个正在等待的线程的视角，内核在做和自己无关的事等价于内核在自己这里轮询。因此我们精巧地模拟出了内核的正确行为。