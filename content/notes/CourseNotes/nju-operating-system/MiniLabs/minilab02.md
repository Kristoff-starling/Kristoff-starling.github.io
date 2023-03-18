---
title: "MiniLab 02: Coroutine Library"
linktitle: MiniLab 02
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

math: true

menu:
    minilabs:
        parent: Contents
        weight: 2
---

本实验的主要目标是实现协程库 (lib coroutines)。很大程度上协程和线程非常相似，一个主要的区别是：线程可以响应中断，然后通过内核的调度器进行切换，而协程如果不主动 yield 则不会切换。

## Coroutine Struct

协程之间共享内存，但各自有独立的运行时栈。笔者定义的协程结构体大致如下：

```c
struct co {
    const char *name;
    void (*func)(void *);
    void *arg;

    enum co_status status;
    struct co      *waiter;
    jmp_buf        context;
    uint8_t        stack[STACK_SIZE];
}coroutine[CO_SIZE];
```

`name` 字段主要用于 debug。`func` 和 `arg` 是目标函数的地址和调用时的参数。`waiter` 存储了指向调用 co_wait() 等待当前协程的“父协程”的指针，当本协程结束时，应该唤醒 (可能存在的) 睡眠的父协程。`context` 存储了切换时的寄存器上下文，在本实验中，我们使用 setjmp() 和 longjmp() 来完成跳转。

## co_new()

co_new() 要做的事情很简单：将所有的协程相关的信息保存到一个结构体变量中即可。我们主要需要思考的是如何开始执行一个新的协程。对于那些调用 co_yield() 中断的协程，我们只需要一个 longjmp() 就可以跳回去，但新的协程的运行时环境的创建要我们自己来操心：

* 我们需要自己完成栈切换并让 PC 指向目标函数的地址。
* 协程对应的函数结束后，我们要指定好返回地址。

笔者最初的实现用內联汇编包办了上面的所有事情：将协程栈地址赋给 %esp/%rsp，用 `jmp` 指令直接跳转，并在栈上提前存储了一个指向 co_exit() 函数地址的“返回地址”来引导协程结束的去向。但后发现其实用 C 语言做一个 wrapper 完成函数跳转更方便：

```c
void wrapper(void *sp, void *entry, uintptr_t arg) {
    // stack switching, in assembly
    ((void(*)(uintptr_t))entry)(arg);
    // coroutine goes here after entry(arg)
}
```

C 编译器会将我们的跳转语句翻译成 call，帮我们填写好返回地址 (不需要额外创建一个 co_exit() 函数了)。我们在內联汇编中需要做的就是将 wrapper() 的栈环境复刻一份到协程的栈上 ( x86_64 甚至只需要改一下 %rsp，因为参数是放在寄存器里的；x86 需要把栈上的参数也搬一份)。

协程结束后，我们应该将其状态设置成 ZOMBIE (等待父协程回收)，并跳转到调度函数。

### Stack Alignment

笔者曾经在 x86_64 中遇到过神秘的段错误，使用 GDB 定位后发现错误语句出现在 printf() 库函数形如

```assembly
movaps %xmm0,0x50(%rsp)
movaps %xmm1,0x60(%rsp)
```

的 SSE 指令。x86_64 对栈有 16 字节的对齐要求 (每个单元是 8 个字节)，这是因为 x86_64 中的一部分寄存器是 16 字节的。出现该错误的根本原因是笔者没有按照 System V ABI 的要求设置好 %rsp：System V ABI (x86-64) 对堆栈对齐的要求是：call 指令执行之前要按 16B 对齐，call 指令执行之后就不按照 16B 对齐了。这其中的原理如下：

* call 指令执行之前，堆栈按 16B 对齐；

* call 指令的语义是：将下一条指令的地址 push 到栈上，然后根据操作数让 PC 跳转到对应的地址执行新的函数。我们向堆栈 push 了一个 8B 的东西 (返回地址)，所以刚执行完 call 指令时堆栈是不对齐的。

* 刚进入一个新函数时，(一般情况下) 编译器生成的头两条指令会是

    ```assembly
    push %rbp
    mov %rsp, %rbp
    ```

    这时我们又向栈上 push 了一个 8B 的东西 (旧 %rbp 值)，因此在正式开始执行函数体时，堆栈又是对齐的了。

在这套流程下，返回地址总是被保存在一个 16B 不对齐的地址上。笔者最初的实现用 jmp 指令直接跳转，且 %rsp 设置成了 16B 对齐，在 16B 对齐的位置手动放了一个返回地址，这样进入新函数 `push %rbp` 之后，堆栈就不对齐了，从而导致错误。

## co_yield()

co_yield() 的代码非常简单：

```c
void co_yield() {
  int val = setjmp(current->context);
  if (val == 0) sched();
}
```

利用 setjmp() 和 longjmp()，我们可以轻松地完成寄存器现场的封存和恢复。setjmp() 在保存寄存器现场时的返回值是 0，longjmp() 跳转回来时可以指定返回值为非零数，这样保存完现场会跳转到调度函数，跳转回来时就可以直接返回协程继续执行。

## co_wait()

co_wait() 要分两种情况：一是子协程已经是僵尸协程，这时候直接释放子协程的资源；二是子协程还不是僵尸协程，这时候要将当前协程睡眠 (状态设置成 WAITING)，子协程结束时会负责唤醒它 (将状态改成 RUNNING)。

## co_init()

我们需要对协程库做一些必要的初始化：比如 main() 函数本身也是一个协程，我们要将 main() 函数作为一个 RUNNING 的协程存储好。初始化函数应当在 main() 函数开始执行之前执行，因此我们需要

```c
__attribute__((constructor)) void co_init() {
    // code
}
```

来修饰 co_init()。