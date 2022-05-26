---
layout: post
date: 2022-03-20
title: "NJU-OS Lecture 10: Application of State Machine"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Compilers and Modern CPU

编译器的作用是将源代码状态机 $S$ 转换为二进制代码状态机 $C$：$C=Compile(S)$。只要两者的可观测行为严格一致，编译器可以随意修改指令执行的顺序，甚至修改指令内容。

<!-- more -->

现代 CPU 本质上也是“编译器”：只要保证硬件状态机的可观测行为一致，它可以乱序执行，可以多发射，还可以将没有依赖关系的指令“并行”地执行，这就是指令级并行 (instruction level parallelism, ilp)。我们可以用一个例子感受超标量处理器 (CPI<1，平均每条指令所需的时钟周期小于 1) 的威力：

```c
#include <stdio.h>
#include <time.h>

#define LOOP 1000000000ul

__attribute__((noinline)) void loop() {
  for (long i = 0; i < LOOP; i++) {
    asm volatile(
      "mov $1, %%rax;"
      "mov $1, %%rdi;"
      "mov $1, %%rsi;"
      "mov $1, %%rdx;"
      "mov $1, %%rcx;"
      "mov $1, %%r10;"
      "mov $1, %%r8;"
      "mov $1, %%r9;"
    :::"rax", "rdi", "rsi", "rdx", "rcx", "r10", "r8", "r9");
  }
}

int main() {
  clock_t st = clock();
  loop();
  clock_t ed = clock();
  double inst = LOOP * (8 + 2) / 1000000000;
  double ips = inst / ((ed - st) * 1.0 / CLOCKS_PER_SEC);
  printf("%.2lfG instructions/s\n", ips);
}
```

该代码的逻辑是在 loop() 中执行 8 条互相没有依赖的指令。即使使用最激进的优化，一次循环至少还有更新循环变量和跳转两条指令，因此一次循环至少 10 条指令。通过计算执行时间和指令条数来估算每秒钟可以执行的指令数。

> **Maybe a bug in GCC?**
>
> 如下代码理论上也可以达到相同的效果。它将变量 i 绑定到寄存器 %rbx 上，从而确保和下面 8 条指令互不冲突。
>
> ```c
> register volatile long i asm("rbx");
> for (long i = 0; i < LOOP; i++) {
>  asm volatile(
>     "mov $1, %rax;"
>     "mov $1, %rdi;"
>     "mov $1, %rsi;"
>     "mov $1, %rdx;"
>     "mov $1, %rcx;"
>     "mov $1, %r10;"
>     "mov $1, %r8;"fang shi |
>     "mov $1, %r9;"
>  );
> }
> ```
>
> 但事实上，编译完成后使用反汇编查看，可以看到 $i$ 被绑定到了 %rax 上。

笔者使用的计算机 CPU 型号如下：

```
Intel® Core™ i7-10710U CPU @ 1.10GHz × 12
```

但运行该程序可以得到每秒钟能执行约 17G 条指令，CPI 远小于 1。这就是指令级并行的威力。

## Tracer

我们总是希望可以在状态机执行的过程中观测状态机的行为，比如了解所有 system call 的执行时长，在某个状态停下来查看寄存器，PC 的值等等，于是我们有了 strace/gdb 等工具。

> **Shell 命令：strace**
>
> 在 strace 后加上 `-T` 选项即可看到执行某个程序过程中所有系统调用的耗时。

我们平时使用 GDB 通常是 step/next/stepi/watchpoint，但基于状态机思想， GDB 可以实现更多炫酷的功能。比如我们可以给某个状态打快照，下次可以直接从快照状态开始执行。再比如我们可以实现反向执行。

### Time-Travel Debugging

让 GDB 实现回退的功能，最直观的想法是保存每个时刻的快照。但运行时刻太多，快照要保存的内容也太多，这样做代价太大。

事实上，一条指令对状态机的影响很有限：它可能只能改变很少的几个寄存器的值或者几个内存地址的值。因此我们可以记录一个初始的状态机，和每条指令执行前后状态机的 diff。这样代价就小得多。想要实现回退，我们只要将增量撤销即可。

GDB 的回退和重放功能对一些 non-deterministic 的程序的调试极其有力。比如如下的程序：

```c
// rdrand.c
#include <stdio.h>
#include <stdint.h>

int main() {
  uint64_t volatile val = 48;
  asm volatile ("rdrand %0": "=r"(val));
  printf("rdrand returns %016lx\n", val);
}
```

rdrand 指令会生成一个随机数，从而使程序每次运行的结果无法预测。打开 GDB 后，使用 `record full` 命令打开记录，这样就可以在适当的时刻使用 `rsi` / `rs` 命令倒退执行，且一旦执行过一次 rdrand 之后，多次倒退回去再执行，变量 val 随机到的值都不会改变。

注：GDB 的反向执行不是万能的。对于一些比较复杂的指令 (例如一些系统调用)，记录状态机增量比较困难，GDB 没有实现这部分功能。

### Record & Replay

如果我们想要重现一个程序执行的行为，我们只需要记录 non-deterministic 的指令执行完成后的状态机状态—— deterministic 的指令是无需记录的：假设 $s_0$ 执行 10000 条确定指令后到达 $s_1$，那么我们只需要记下 $s_0$ 和 10000 即可重放这段执行。

对于一个单线程的程序，我们需要记录的非确定事件包括系统调用，rdrand 等非确定指令等。Mozilla 的 rr 工具可以帮助我们记录并重放一个程序的执行过程。这对我们调试一些非确定程序的段错误极其有帮助。

> **Mozilla 的 rr 工具**
>
> 使用命令 `rr record ./exec` 即可记录运行程序 exec 的整个过程。使用 `rr replay` 命令即可重放整个过程。rr 会打开一个 GDB，我们可以自由地打断点并调试之前记录的执行过程。
>
> rr 工具要求 perf_event_paranoid 的值小于等于 1，可以使用命令
>
> ```bash
> sudo sysctl kernel.perf_event_paranoid=1
> ```
>
> 修改当前地 perf_event_paranoid 的值。

对于一个单处理器的操作系统，我们需要记录的非确定事件包括 I/O 操作，中断等等。有了这些记录我们便可以重放操作系统启动和运行的全过程。

## Profiling

想要做好性能优化，我们需要知道时间花在了哪里，什么对象占用了空间……这要求我们对状态机的执行进行真实监测。但我们的监测不能太频繁，不能干扰到程序的正常工作。一个好的思路是每隔一段事件暂停程序，观察并记录状态机的信息，这样便可以得到统计意义的性能摘要。

Linux perf 工具可以为我们提供一个程序运行的性能报告。以之前的 `ilp-demo.c` 为例，输入命令 `perf stat ./ilp-demo`，我们可以得到如下的精简报告，其中包括了上下文切换，缺页异常分支预测等多种信息：

```
Performance counter stats for './ilp-demo':

              0.56 msec task-clock                #    0.449 CPUs utilized          
                 0      context-switches          #    0.000 K/sec                  
                 0      cpu-migrations            #    0.000 K/sec                  
                55      page-faults               #    0.098 M/sec                  
           975,780      cycles                    #    1.731 GHz                    
           751,657      instructions              #    0.77  insn per cycle         
           146,602      branches                  #  260.113 M/sec                  
             5,119      branch-misses             #    3.49% of all branches        

       0.001255936 seconds time elapsed

       0.001289000 seconds user
       0.000000000 seconds sys
```

我们还可以使用 `perf recode ./exec` 和 `perf report` 查看详细的报告。

实际工程中的大部分情况满足“二八定律”：80%的时间消耗在非常集中的几处代码。因此用好 profiling 工具，有的放矢进行优化才是科学的方式。

## Model Checker

我们之前使用 model checker 检查并发 bug。事实上 model checker 可以检查更多我们想要的东西。比如如下程序：

```c
u32 x = rdrand();
u32 y = rdrand();
if (x > y)
  if (x * x + y * y == 65)
    bug();
```

我们想知道理论上 bug 是否可能会被触发。一个朴素的想法是对每种 x 和 y 的可能取值建立状态，但这样状态会太多，无法检查。更高效的 model checker 会将相似状态“合并”，或者引入符号计算——上述例子中，对于 rdrand()，我们可以给 x，y 赋值 uint32，if 语句的约束条件也一并写入状态机，最后调用约束求解器 (SMT Solver) 即可检查 bug 是否会被触发。