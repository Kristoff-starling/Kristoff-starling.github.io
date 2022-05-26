---
layout: post
date: 2022-05-04
title: "NJU-OS Lecture 22: OSLab Speedrun"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## L1

一把大锁保平安策略：全局用一把锁，把所有需要保护的代码都用 `atomic {}` 框起来。

```c
#define atomic \
	for (int __i = (lock(), 0); i < 1; unlock(), __i++)
```

<!-- more -->

这个 for 循环的语义如下：

```c
lock();
__i = 0;	// C语言逗号表达式总是取后面的值
check i < 1 => yes
body
__i++;
unlock();
check i < 1 => no, break
```

使用这个 `atomic {}` 我们要保证程序不能在 atomic 内部 break 或 return。

> **Programming Philosophy**
>
> 在书写不是 performance bottle neck 的代码时，我们应该尽可能将代码写得简单，尽量不要写奇技淫巧，以牺牲可读性、可维护性为代价换取微不足道的性能。此外，我们应该在程序中添加足够的 assertion 来尽早抓取到 bug。

## L2

一个允许嵌套的大锁：

```c
int locked;
void lock() {
    int c = cpu_current();
    bool i = ienabled();
    
    iset(false);
    nest[c]++;
    if (nest[c] == 1) {
        intena[c] = i;
        while (atomic_xchg(&locked, 1));
    }
}
void unlock() {
    int c = cpu_current();
    
    nest[c]--;
    if (nest[c] == 0) {
        atomic_xchg(&locked, 0);
        if (intena[c]) iset(true);
    }
}
```

> **Programming Philosophy**
>
> 在写代码的过程中，在一行里写很多表达式，让一行变得很长是一种 bad practice。我们可以适时地用一些临时变量保存一些值，这样可以大幅提高代码的可读性和可维护性。

线程创建：

```c
struct task {
    Context ctx[4];
    int nc;
    int stk[STK_SZ];
};
struct percpu {
    struct task *avail[NPROC];
    int n, c;
}percpu[MAX_CPU];
struct task *kcreate(void *entry, void *arg, int cpu) {
    struct task *t = kalloc(sizeof(struct task));
    atomic {
        t->ctx[0] = *kcontext((Area){&t->stk, &t->stk + STK_SZ}, entry, arg);
        t->nc = 1;
        percpu[cpu].avail[percpu[cpu].n++] = t;
    }
    return t;
}
```

一个最简单的 round-robin 调度器：保存当前线程的上下文，切换到下一个线程，返回目标线程的上下文。

```c
#define CPU (&percpu[cpu_current()])
#define current (CPU->avail[CPU->c])

Context *trap(Event ev, Context *ctx) {
    current->ctx[current->nc++] = *ctx;
    
    atomic {
        CPU->c = (CPU->c + 1) % (CPU -> n);
    }
    assert(!ienabled());
    
    return &(current->ctx[--current.nc]);
}
```

信号量最简单的语义就是：在没有资源的时候 yield()。

```c
struct semaphore {
    int count;
}
void sem_wait(sem_t *sem) {
    bool succ = false;
    while (!succ) {
        atomic {
            if (sem->count > 0) {
                sem->count--;
                succ = true;
            }
        }
        if (!succ) yield();
    }
}
void sem_post(sem_t *sem) {
    atomic {
        sem->count++;
    }
}
```

> **Programming Philosophy**
>
> 上述的信号量实现是一个比较基础的实现。一个性能更好的实现是调用 sleep()，当前条件不满足的话就释放自旋锁并将自己加入一个等待队列，但这种实现就会引入更多可能错误的点。事实上，我们可以为各个模块都准备一份 functional 的代码 (非常简单，保证正确，类似于模型) 和一份注重性能的代码。当项目出现 bug 的时候，我们就将一个一个模块换回 dummy 的实现，从而快速定位问题。

## L3

ucreate 和 kcreate 结构差不多，这次我们使用 ucontext 创建初始上下文。我们假设代码段总是被加载到地址空间的开头。此外，现在 task 结构体中还需要加入一个地址空间 `AddrSpace as`。

```c
task_t *ucreate(int cpu) {
    task_t *t = kalloc(sizeof(task_t));
    atomic {
        protect(&t->as);
        t->ctx[0] = *ucontext(&t->as, (Area){&(t->stk), &(t->stk)+STK_SIZE}, &t->as);
        t->nc = 1;
        percpu[cpu].avail[percpu[cpu].n++] = t;
    }
    return t;
}
```

在 os_trap 中，我们要检查更多的事件，例如 page fault，syscall 等：

```c
switch (ev.event) {
    case EVENT_PAGEFAULT: {
        pagefault(ev, ctx);
        break;
    }
    case EVENT_SYSCALL: {
        assert(current->nc == 1);
        current->ctx[0].GPRx = syscall(ctx);
        break;
    }
    case EVENT_ERROR: {
        panic();
        break;
    }
    default: break;
}
```

缺页异常我们的处理是在虚拟地址空间中为当前虚拟地址映射一个物理页面。此外如果缺少的是第一个页面 (即 init 进程的代码段) 我们要把代码段拷贝进来。

```c
void pgmap(task_t *t, void *va, void *pa) {
    t->va[t->np] = va;
    t->pa[t->np] = pa;
    t->np++;
   	map(&t->as, va, pa, MMAP_READ | MMAP_WRITE);
}
void pagefault(Event e, Context *c) {
    atomic {
        AddrSpace *as = &(current->as);
        void *pa = kalloc(as->pgsize);
        void *va = (void *)(e.ref & ~(as->pgsize - 1L));
        
        if (va == as->area.start) memcpy(pa, _init, _init_len);
		
        pgmap(current, va, pa);
    }
}
```

处理各种系统调用：

```c
int syscall(Context *c) {
    int ret = 0;
    ise(true);
    switch (c->GPRx) {
        case SYS_kputc: {
            putch(c->GPR1);
            break;
        }
        case SYS_sleep: {
            uint64_t wakeup = io_read(AM_TIMER_UPTIME).us + 1000000L * c->GPR1;
            while (io_read(AM_TIMER_UPTIME).us < wakeup) yield();
        }
        case SYS_fork: {
            atomic {
                struct task *t = ucreate(cpu_current());
                
                uintptr_t rsp0 = t->ctx[0].rsp0;
                void *cr3 = t->ctx[0].cr3;
                
                t->ctx[0] = *c;
                t->ctx[0].rsp0 = rsp0;
                t->ctx[0].cr3 = cr3;	// 子进程的页表和内核栈不能复制父进程
                t->ctx[0].GPRx = 0;		// 子进程的返回值应该是0
                
               	for (int i = 0; i < current->np; i++) {
                    int sz = current->as.pgsize;
                    void *va = current->va[i];
                    void *pa = current->pa[i];
                    void *npa = kalloc(sz);
                    memcpy(npa, pa, sz);
                    pgmap(t, va, npa);
                }
            }
        }
    }
    iset(false);
}
```