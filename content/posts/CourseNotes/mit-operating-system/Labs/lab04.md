---
title: "MIT-6.S081 Lab 04: Traps"
linktitle: 'Lab traps'
type: docs
draft: false
featured: false

math: true

menu:
    mitos-labs:
        parent: Contents
        weight: 4
---

## Progress

- [x] RISC-V assembly (easy)
- [x] Backtrace (moderate)
- [x] Alarm (hard)

## RISC-V assembly (easy)

> **Q: Which registers contain arguments to functions?  For example, which register holds 13 in main's call to `printf`?**  

RISC-V 中，函数调用的参数按顺序存放在 `a0` `a1` `a2` `a3` …… 中。以 printf() 为例，13 是它的第三个参数，所以存放在 `a2` 寄存器中。

> **Q: Where is the call to function `f` in the assembly code for main? Where is the call to `g`?  (Hint: the compiler may inline functions.)**  

汇编代码中没有直接调用函数 f() 和 g()，因为 f() 和 g() 是非常简单的函数，编译器对其做了优化，直接将 `f(8)+1` 的结果 12 写入了寄存器 `a1`。

> **Q: At what address is the function `printf` located?**  

printf() 函数的地址为 0x628。可以在 `call.asm` 中直接搜索到，也可以通过 main() 函数的汇编代码

```assembly
30:	00000097          	auipc	ra,0x0
34:	5f8080e7          	jalr	1528(ra) # 628 <printf>
```

看出 printf() 函数的地址。

> **Q: What value is in the register `ra` just after the `jalr` to `printf` in `main`?**  

`ra` 寄存器中存储的应该是 printf() 函数的返回地址。根据 RISC-V 手册中 `jalr` 指令的定义，pc+4 的值会被写入 `ra` 寄存器。因此当前 `ra` 的值为 0x38。

> **Q: Run the following code.**      
>
> ```c
> 	unsigned int i = 0x00646c72;
> 	printf("H%x Wo%s", 57616, &i);
> ```
>
> **What is the output? [Here's an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html) that maps bytes to characters.**     
>
> **The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?**
>
> **[Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).**    

输出的结果是：

```
He110 World
```

第一，十进制数 57616 转换成十六进制是 E110H。使用 "%x" 打印时，十六进制中的字母都是小写字母，所以第一处打印结果为 e110。

第二，变量 i 的地址被传给了 printf()，printf() 会将该地址处的数据按照字符串类型进行解读，即每个字节按照 ASCII 码翻译成字符进行输出。RISC-V 是小端机器，因此地址 &i 处往后若干个字节的内容为

| &i                   | &i + 1               | &i + 2               | &i + 3                |
| -------------------- | -------------------- | -------------------- | --------------------- |
| 72 $\rightarrow$ 'r' | 6c $\rightarrow$ 'l' | 64 $\rightarrow$ 'd' | 00 $\rightarrow$ '\0' |

遇到结束符即停止输出，因此第二处打印结果为 rld。

如果 RISC-V 是大端机器，为了使第二处输出结果不变，i 的值应该修改为 0x726c6400。第一处的 57616 不需要修改，因为只要存和取都遵循大端方式，数据的值就不会改变。

> **Q: In the following code, what is going to be printed after `'y='`?  (note: the answer is not a specific value.)  Why does this happen?**       
>
> ```c
> 	printf("x=%d y=%d", 3);
> ```

`y=` 后的内容是进入 printf() 函数后 `a2` 寄存器中的值。由于这里的 printf() 函数少了一个参数，`a2` 寄存器中的值是 UB，输出结果会是一个无法预料的随机数。

## Backtrace (moderate)

要实现的 Backtrace 功能和 GDB 中的 backtrace 基本相同：根据栈帧内容回溯函数调用的过程。

首先需要获取当前函数的栈帧。sp 寄存器指向栈顶，但栈帧的大小是不确定的，我们需要的是栈底指针 fp (即 s0 寄存器)。可以通过内联汇编的方式来读取 fp 寄存器的值。

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : (=r) (x));
  return x;
}
```

由于现在处在内核态，当前的栈帧存储在上一个用户进程的内核栈上。内核栈大小为一个页面，可以通过 `PGROUNDDOWN(fp)` 和 `PGROUNDUP(fp)` 来分别获得内核栈的栈顶地址和栈底地址。我们的函数回溯追踪只在内核栈范围内，因为更远的回溯会到用户栈上。

根据 RISC-V 的栈帧格式，fp 指针实际上指向的是上一个栈帧的底部 (课程中 TA 的说法有误)。在 RV64 中，fp-8 的位置存储了当前函数的返回地址，这个地址理应在当前函数的 caller 函数范围内，打印这个地址就相当于追踪到了上一个函数。fp-16 的位置存储了上一个栈帧的 fp 的内容。将这个值赋给 fp 即可实现顺着栈帧向上爬的功能。

> **关于 printf() 的输出不能得到正确的结果**
>
> 笔者起初使用 printf() 的 `%x` 来输出返回地址，但得到的结果出入很大。xv6 中的 printf() 不是调用的 C 库函数，而是自己实现的。阅读 printf() 的源码可以看到，`%x` 输出十六进制数只能输出 int 范围内的。要打印一个地址应该使用 `%p` 模式来输出。

## Alarm (hard)

我们需要实现两个系统调用

```c
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
```

`sigalarm()` 负责在当前进程中打开一个定时器，每过 ticks 个周期就执行一次 handler 函数。为了实现这个功能，我们需要在 proc 结构体中添加一些字段：nticks 负责存储当前的周期，handler 负责存储回调函数的地址 (用户虚拟地址)，curticks 负责记录自上一次调用 handler 之后，已经过了多少个周期。在 sys_sigalarm() 函数中，我们应当更新这些字段。

此处的周期指的不是 CPU 的时钟周期，而是指时钟中断的周期。因此在 usertrap() 函数中，每次判断到 `which_dev == 2 `，即时钟中断到来，则 `curticks++`，若 curticks 达到了 nticks，则下一次返回用户态应转移到回调函数。sepc 寄存器决定了返回用户态时的地址，因此只要将 handler 赋给 p->trapframe->epc 即可。

回调函数的最后一定会使用 `sigreturn()` 系统调用，该系统调用负责回到原本用户程序的中断处继续执行。触发了回调函数的这次中断走过的路程应该是这样的：

​		用户程序 $\overset{时钟中断}{\rightarrow}$ 内核 $\rightarrow$ 回调函数 $\overset{系统调用}{\rightarrow}$ 内核 $\rightarrow$ 用户程序

在触发时钟中断时，用户程序的上下文会被保存在 p->trapframe 中，但从回调函数再进入内核的时候，trapframe 里的上下文会被覆盖。要想对用户程序完全透明地执行这个过程，我们要在执行回调函数之前将原始上下文保存起来。因此我们需要在 proc 结构体中再定义一些变量用来保存原始上下文。笔者使用的一种简单的方法是定义一个 trapframe 结构体 u_trapframe，在进入回调函数之前将 p->trapframe 里的内容写到 u_trapframe 里。

此外 test2 中还测试了一种问题：我们在进入回调函数的时候，应该关闭 sigalarm() 设置的定时器。否则如果回调函数执行时间过长，在执行回调函数的过程中某一次时钟中断又触发定时器调用回调函数，就会陷入死递归。因此我们在 proc 结构体中还需要一个变量来记录当前是否是从回调函数中陷入内核态的。这个状态应该在前一次准备进入回调函数时置 1，在 sigreturn() 中清零。当该变量置 1 时，时钟中断不对计时器进行检查。