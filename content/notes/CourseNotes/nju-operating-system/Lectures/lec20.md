---
title: "Lecture 20: Scheduling"
linktitle: '20: 处理器调度'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 20
---

处理器调度问题的简单假设 (1960s)：

* 系统中有一个处理器
* 系统中有多个进程/线程共享 CPU
    * 包括系统调用 (进程/线程的一部分在内核代码中)
    * 偶尔会等待 I/O 返回，不使用 CPU。

## Policy

### Round-Robin

按照时间片轮转，轮流运行所有的进程/线程。

问题：假设当前有一个前台的 vim 进程和很多计算型的后台进程，如果使用 round robin，很容易出现前台 vim 进程很卡的现象 (vim 进程一旦被切出去了，就要等一整轮才能获得一个时间片)。

我们希望引入某种“优先级”，使得和用户交互比较频繁的前台进程可以获得更高的调度优先级，优化用户的使用体验。

### UNIX Niceness

UNIX 的 niceness 是一个 `-20 ... 19` 的整数，nice 值越高的进程越”和善“，越容易让出 CPU。

基于优先级的调度策略：

* 在一些实时的操作系统中 (尤其是物理相关，比如火箭/机械手 etc.)，CPU 会完全根据优先级，每次选择 niceness 最低的进程执行。这样的做法是有道理的：比如火箭要运行应急程序，我们自然不希望时钟中断后收集火箭运行数据的进程插入进来执行。

* 在桌面操作系统中，让“坏人”霸占 CPU 的做法不太可取。在 Linux 中，niceness 相差 10，CPU 资源获得率相差大约 10 倍。

    > 我们可以做一个实验来验证这一点：
    >
    > ```bash
    > taskset -c 0 nice -n 19 yes > /dev/null &
    > taskset -c 0 nice -n  9 yes > /dev/null &
    > ```
    >
    > 其中 taskset 命令可以指定让一个进程在某个 CPU 核上执行。创建这两个后台进程后，我们可以通过 top 来观测它们的 CPU 占用情况。
    >
    > // office hour: 为什么我使用这两个指令创建出的进程 nice 值是 14 和 19？...

### MLFQ (Multi-level Feedback Queue)

仍然考虑 round-robin 在遇到一堆计算密集型线程和一个交互型线程时的问题。我们现在考虑：能否让操作系统观测进程的行为，然后动态地调整进程的优先级？这就是 MLFQ 的基本思想。

操作系统观测每个进程对时间片的使用情况：如果一个进程每次都用满整个时间片，那么它是一个“坏进程”，我们就适当降低它的优先级；如果一个进程经常 sleep()，等待 I/O etc. CPU 占用比较小，那么它是一个“好进程”，我们就适当增加它的优先级。我们动态调整进程的优先级，为每种优先级创建一个 round robin 的队列。

当然，一个纯粹的”坏进程”可以通过如下的方式来欺骗这样的策略：

```c
while (1) {
    compute();
    if (时间片快用完了) usleep(1);	
}
```

此外，如果像 Vim 这样的高优先级进程过多，后台的计算进程可能就会几乎卡死；进程的“好坏”其实是动态变化的，不能“一棍子打死”等等。操作系统的设计者会试图补救这些问题，比如 priority boost 策略会每隔一段时间把所有进程的优先级拉平，使得进程有“重新做人”的机会。

如果我们考虑进程之间的通信，这个问题就会变得更加复杂：假设进程中有 producer/consumer，以及其他的 `while(1)` 线程，在 round-robin 中 producer/consumer 会频繁让出 CPU，这导致 `while(1)` 获得了过多的运行时间；在 MLFQ 中，producer/consumer 因为频繁让出 CPU 会获得高优先级，但这种让出并不是出于人机交互的目的——producer/consumer 组合起来仍然是一个纯计算任务，因此它获得高优先级也是不合理的。

### CFS (Complete Fair Scheduling)

完全公平调度的指导思想很简单：让每个进程公平地享用 CPU。因此操作系统内核会记录每个进程运行的时间，每次调度器选择当前执行时间最少的进程执行。

为了避免落入 round robin 的结局，CFS 通过类似于“变速齿轮”的方法实现优先级：每个进程实际上享用的是相同的虚拟运行时间 (vruntime)，但不同进程的虚拟运行时间和物理运行时间的比例不同：优先级高的进程一个单位的 vruntime 对应更长的物理时间，反之亦然。这种方式类似于变速齿轮是因为它实际上“欺骗”了进程：进程能直接感受到的物理时间是虚拟的。进程其实可以通过其他方式感知到“不对劲”：比如 "我 1s 怎么只执行了 1M 的指令？" 但现在的进程还没有这样的能力。

#### CFS Complexity: New Process/Thread

考虑如下一段代码：

```c
#include <stdio.h>

int main () {
    setbuf(stdout, NULL);
    int pid = fork();
    if (pid < 0) perror("fork");
    else if (pid == 0)
        for (const char *s = "child"; *s; s++) putchar(*s);
    else
        for (const char *s = "parent"; *s; s++) putchar(*s);
    return 0;
}
```

在较新版本的 linux 内核中，多次运行上述程序会看到几乎总是 parent 先被打印。这是因为现在的内核是 parent first 的，曾经的内核版本是 child first，这其中存在一个权衡问题：绝大部分情况下 fork() 执行完会紧接着执行 execve()，我们的 fork() 是 copy-on-write 的，如果先执行子进程，有很大概率 execve() 会销毁原先的页表，这样父进程执行时就不需要额外的复制操作；但先执行子进程意味着当前 CPU 的 cache, TLB 等需要被全部刷新，这又会有一个立即可见的额外代价。

除了 parent/child first 问题，另一个问题是：我们的 CFS 应该为这个新进程创建怎样的 vruntime? 早期的版本会给子进程较少的 vruntime，也就是让它有机会稍微多执行一点。但这个策略导致恶意程序可以疯狂 fork 子进程来占据 CPU，因此现在的内核代码中子进程继承父进程的 vruntime。

```c
// linux kernel
static void task_fork_fair(struct task_struct *p) {
    ...
    if (curr) {
        update_curr(cfs_rq);
        se->vruntime = curr->vruntime; // <------------ here
    }
}
```

#### CFS Complexity: I/O

考虑如下场景：有一个进程等待了 1min 的 I/O 操作，这段时间内它一直处于 blocked 的状态，不会被调度上 CPU。那么 I/O 操作结束后，该进程的 vruntime 会显著低于别的进程。从而未来的相当长时间内 CPU 会被这个 I/O 进程独占，这显然不是我们想看到的情况。

操作系统理应在 I/O 进程结束等待后将其 vruntime 补齐到一个基准点，这个基准点到地设置在哪里又是很有讲究的事。

#### CFS Complexity: Integer Overflow

在一个需要长时间运行的系统中，我们即使用 `uint64_t` 来存储 vruntime 也要面临溢出的问题。这意味着我们不能用 vruntime 的绝对大小来比较两个进程的优先级。

Linux 中采取的解决方案是：保证任意时刻系统中最大的 vruntime 和 最小的 vruntime 之差不超过数轴的一半 (i.e. `UINT64_MAX / 2` )。这样我们可以用比较相对大小的方式来解决问题：

```c
// linux kernel
bool less(uint64_t a, uint64_t b) {
    return (int64_t)(a - b) < 0;
}
```

我们可以考虑一下 a 是很小的正数 (溢出了一轮)，b 是很大的正数 (还没溢出) 的情况，此时应有 $a>b$。$a-b$ 的结果是小于 0 的，但由于这个值大于 `INT64_MAX`，所以强制转换成 `int64_t` 后会下溢出变成正的，从而正确实现功能 (非常 tricky 的代码)。

#### CFS Complexity: Implementation

我们要实现一个数据结构，支持高效的插入、删除、找最小值等。Linux 采取了主流的红黑树设计，但我们的数据结构要保证并行状态下的正确性，为了效率还不能上大锁……

## The First Bug on Mars

在实时操作系统中，高优先级意味着必须先被调度。简单的调度处理可能会出现优先级反转的情况：

```c
void high()   {sleep(1); mutex_lock();}
void medium() {while (1);}
void low()    {mutex_lock();}
```

假设低优先级进程先执行，获得了锁；然后 medium() 到来，将 low() 踢出 CPU，low() 带着锁进入了睡眠；这时 high() 来了但发现无法获得锁——因为锁在 low() 手里。high() 因为锁的语义等待 low() 是可以接受的 (锁可能很快就被释放)，但 low() 在等待 medium()，这导致 high() 间接等待了一个与自己毫无关系的进程 medium()，这是不可接受的。

这个 bug 在第一辆火星车上真实地发生过，这导致了火星车系统的重启。解决优先级反转地方法通常有优先级继承 (priority inheritance)/优先级提升 (priority ceiling)，简单来说，当 high() 因为锁资源等待时，low() 的优先级可以暂时提升到和 high() 一样高，这样很快 low() 就可以把 medium() 踢下 CPU 运行，释放了锁 high() 就可以正常执行了，low() 的优先级也恢复最低。

## Multi-core Scheduling

多核处理器上的调度处于一个两难的境地：

* 如果简单地将线程分配到处理器，那么容易出现“一核有难他核围观”的局面。
* 如果简单地将线程安排到一个空闲处理器上，那么之前的 cache/TLB 就全部白给了。

### Complexity: Multi-user

假设用户 A 和 B 共用一台服务器，A 的程序只能单线程，而 B 的程序可以 1000 个线程，那就会出现 A 只获得了很少的资源的情况。糟糕的是，A 不能通过提高自己线程的优先级的方法来解决这个问题：在 Linux 中，非特权用户只能提升自己的 nice 值，不能降低。

Linux 采用的解决方案是 namespaces control groups，大致思想是创建了“操作系统中的操作系统”，将属于一个用户的进程归成一组管理。

### Complexity: Big.LITTLE and Energy Consumption

现在的 CPU 有大小核的概念：大核频率高，小核频率小，我们在调度的时候不得不将这些因素考虑进来。此外，现在的软件可以配置 CPU 的工作模式：CPU 的频率越低，它的延迟越高，但能耗越小，吞吐量越高。那么如何根据不同类型的任务调整 CPU 工作模式以及调度也是非常复杂的问题。

### Complexity: Non-Uniform Memory Access

共享内存在某种程度上只是一个假象。L1 Cache 花了巨大的代价才让各个 CPU 看到了共享的内存。多个线程在同一个/不同的核上跑，效率可能有很大的区别。

在实际情况下，连”核心越多，速度越快”的假设都可能是错的。以 `sum-atomic.c` 为例，如果我们用命令

```bash
tastset -c 0 ./sum-atomic
```

来让 `sum-atomic.c` 的所有线程只工作在一个核上，会发现运行时间竟小于多个核！调度器把程序当作黑盒的假设可能是不对的。

### Complexity: CPU Hot-plug

现在的 CPU 支持热插拔，可能运行一会儿会发现多了一个或者少了一个……

---

调度是一个非常复杂的问题，将这个任务完全丢给操作系统的调度器是不合理的。一种可能的想法是：让程序提供一些 scheduling hints 来指导调度行为。



