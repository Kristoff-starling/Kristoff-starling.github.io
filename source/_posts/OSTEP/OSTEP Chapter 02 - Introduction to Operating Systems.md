---
layout: post
date: 2022-02-50
title: "OSTEP Chapter 02: Introduction to Operating Systems"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

操作系统：为上层多个应用程序的正常运行提供支持。操作系统实现这一目的的重要方法是虚拟化：将硬件资源抽象成简单易用的虚拟资源。为了方便上层应用程序使用这些资源，操作系统会提供数百个系统调用。从这个角度来看，操作系统也很像一个标准库。

<!-- more -->

## 2.1 Virtualizing The CPU

[cpu.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2002%20-%20intro/cpu.c) 的作用是每过一秒钟输出一次字符串。`Spin(1)` 表示等待一秒，其实现在 [common.h](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2002%20-%20intro/common.h) 中，其中 GetTime() 是对系统调用 gettimeofday() 的进一步封装，返回系统启动以来的秒数。Spin() 会不断调用 GetTime()，直到间隔时间达到输入值 howlong。

键入命令 `./cpu A & ; ./cpu B & ; ./cpu C & ; ./cpu D &`，可以看到A,B,C,D 交替输出，仿佛各自独占了 CPU。操作系统通过虚拟化 CPU 的方式，给上层应用程序一种系统中有很多很多个 CPU 的假象。至于多个进程谁在真正的物理 CPU 上运行，这取决于操作系统的调度策略。

> **为什么程序会不停输出，按 `Ctrl+C` 也无法停止？**
>
> 上文中，用分号隔开各个命令表示这些命令都要执行。它和 `&&` 的区别在于：`&&` 要求第一个命令执行成功才会执行第二个命令，而 `;` 不论第一个命令是否成功都会执行第二个命令 (可以用 `gcc notexist.c ; ls` 和 `gcc notexist.c && ls` 做实验验证)。
>
> 在命令最后加上 `&` 表示将这个进程放到后台执行， `Ctrl+C` 的意义是中断所有正在运行的前台进程，因此该组合键无法终止后台运行的进程。如果某个命令所需要的执行时间很长，可以用 `&` 将其放在后台执行，从而终端界面仍然可以继续操作。在使用 `&` 时，bash 会提示分配给该任务的进程号。如果想要结束后台任务，可以使用命令 `kill pid` 。
>
> 这里我们使用 `&` 的原因在于：用 `;` 分隔的若干命令总是会依次执行，即第一个执行结束才会执行第二个。而  cpu 程序中有一个死循环，为了观测“并发”，我们必须让四个任务同时运行起来。

## 2.2 Virtualizing Memory

物理内存本身没有任何特殊之处，就是一个大数组，每个位置有一个物理地址。

[mem.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2002%20-%20intro/mem.c) 会调用 malloc() 分配一个地址，并不断向该地址写入内容。如果我们使用 `./mem 1 & ; ./mem 1 &`  命令，我们会发现两个进程分配的地址是一样的，然而两个进程都在完好地运行。这是因为每个进程都有自己的虚拟地址空间，输出的地址是进程的虚拟地址，不同进程的相同虚拟地址会指向不同的物理地址。操作系统负责虚拟化内存，保证每个进程只能访问自己的地址空间。

> **为什么我使用上述命令无法复现，两个地址不一样？**
>
> Linux 默认使用了地址空间随机化 (Address Space Layout Randomization, ASLR) 技术。ASLR 是一种针对缓冲区溢出攻击的安全保护技术，通过对堆、栈、共享库映射等布局的随机化增加攻击者预测目的地址的难度，关于利用固定地址进行攻击的方法可以见[这篇文章](https://kristoff-starling.github.io/2022/02/28/Stack-smashing%20Attack%20-%20An%20Introduction/)。在 `/proc/sys/kernel/randomize_va_space` 文件中，我们可以查看当前 ASLR 是否打开：0 表示关闭，1 表示对于部分函数打开，2 表示完全打开。
>
> 为了暂时关闭 ASLR 以复现上述情境，我们可以使用命令：
>
> ```bash
> setarch `uname -m` -R /bin/zsh
> ```
>
> 其中 `uname -m` 命令会输出机器架构，该参数可以省略；`-R` 参数 (对应长参数 `--addr-no-randomize`) 表示关闭 ASLR。执行该命令会暂时打开一个新的 shell (默认 `/bin/sh`，可以通过最后一个参数指定其他 shell)，该 shell 和其子进程会在关闭 ASLR 的情况下执行命令 (退出该 shell 后，一切恢复)。
>
> 另外一种方法是使用命令
>
> ```bash
> sysctl -w kernel.randomize_va_space=0
> ```
> 机器重启后该改变会消失。如果想要永久性的关闭 ASLR，可以将 `kernel.randomize_va_space=0` 写到 `/etc/sysctl.conf` 中。
>
> 如果使用 GDB 调试，可以通过命令 `set disable-randomization off` 关闭 ASLR。

## 2.3 Concurrency

[threads.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2002%20-%20intro/threads.c) 创建了两个线程，创建线程调用的函数是 Pthread_create()，它在 `common_threads.h` 中定义，是对 POSIX 线程库的简单封装：

```c
#define Pthread_create(thread, attr, start_routine, arg) assert(pthread_create(thread, attr, start_routine, arg) == 0);
#define Pthread_join(thread, value_ptr)                  assert(pthread_join(thread, value_ptr) == 0);
```

让两个线程各执行 N 次 +1 操作，可以观测到结果不是 2N。这主要是因为核心语句 `counter++` 会被编译成三条汇编语句：

```assembly
mov &counter, register
add $1, register
mov register, &counter
```

由于原子性的丧失，可能出现 race condition。

## 2.4 Persistence

数据的持久性是一个重要的话题。内存中存储的数据是不稳定的——机器一旦断点，DRAM 中的信息就会丢失。我们需要硬盘来存储长期数据，硬盘在现代系统中被看作 I/O 设备的一部分，管理硬盘信息的软件称为文件系统。

操作系统对于硬盘的抽象和 CPU，内存不同。我们针对 CPU 抽象出了进程，对于内存抽象出了虚拟地址空间，其目的都是为了让某一个程序“独占”整个计算机资源。而硬盘上的数据理应让各个程序共享，比如编辑器创建了一个文件，然后编译器负责编译它，接着加载器负责加载、运行这个程序。操作系统通过系统调用来抽象硬盘：将复杂的硬件读写操作封装成简单的接口。

在 [io.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2002%20-%20intro/io.c) 中我们使用了 open() 和 write() 系统调用来创建，写入文件。这些系统调用底层的实现非常复杂 (比如为了性能的提升，读写都有缓冲区)，但上层使用这些 API 非常简便。OS 在这里充当了标准库的角色。

## 2.5 Design Goals

操作系统应当努力追求的目标：

* 高性能；
* 保护/隔离：应用程序之间不能互相“伤害“；
* 可靠：操作系统一旦崩溃，在其上运行的所有应用程序都无法使用，因此操作系统必须是非常可靠的软件。
* ……
