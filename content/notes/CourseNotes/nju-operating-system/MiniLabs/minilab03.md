---
title: "MiniLab 03: Syscall Profiler"
linktitle: MiniLab 03
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

math: true

menu:
    minilabs:
        parent: Contents
        weight: 3
---

该实验的主要目标是利用 strace 系统调用实现一个性能监测器，实时输出当前进程运行耗时排名前 5 的系统调用。

## Overview

该实验的大体思路为：我们需要 fork 出一个子进程，让子进程用 strace 运行目标程序，并把 strace 的结果通过管道输送给父进程。本实验只允许使用 execve() 函数，不允许使用带有 p 的库函数，这意味着我们必须手动解析环境变量，将所有的路径一一加在 strace 的前面并调用 execve()，直到找到 strace 的确切位置为止 (如果成功执行，函数不会返回)。父进程负责不断从管道中读取 strace 的输出，解析出系统调用名和消耗的时间并添加到数据结构中，每隔一段时间输出一次。

以下主要记录实验中遇到的一些有趣的细节。

## dup() vs dup2()

创建管道以后，我们会有将子进程的标准输出和父进程的标准输入连接到管道的写口/读口的需求。我们可能会写出这样的代码：

```c
close(0);        // close stdin
dup(pipefd[0]);
```

dup() 总是会挑选当前最小的未使用文件描述符，因此关闭了 stdin 后，dup() 会将 0 号文件描述符复制为管道写口的文件描述符。但这其实是一个 bad practice：在多线程的程序中，如果 close() 和 dup() 之间被插入了其他执行流，我们不能保证 dup() 执行时最小的未使用文件描述符仍然是 0。因此我们应该使用函数 dup2()：

```c
dup2(pipefd[0], 0);
```

dup2() 系统调用的声明为

```c
int dup2(int oldfd, int newfd);
```

它和 dup() 的含义基本相同，不同之处在于我们可以指定将 oldfd 复制到 newfd 上而不是“最小的未使用文件描述符”。如果 newfd 对应的文件处于打开状态，dup2() 会将其先关闭再使用该描述符。dup2() 的好处在于关闭文件和使用 newfd 的过程是**原子**的 (手册保证了这一点)。

## Time Information Matching

笔者尝试使用了正则表达式来匹配 strace 输出信息中和运行时间相关的有效内容。`regex.h` 库中提供了和正则表达式相关的库函数。一些比较有用的函数罗列如下：

```c
int regcomp(regex_t *preg, const char *regex, int cflags);
```

该函数负责将一个正则表达式编译成一个 regex_t 类型的东西。

```c
int regexec(const regex_t *preg, const char *string, size_t nmatch, regmatch_t pmatch[], int eflags);
```

regexec() 负责匹配。`preg` 是编译好的正则表达式，`string` 是匹配的串。`nmatch` 是匹配次数。pmatch[] 数组会存储 nmatch 个匹配结果，其中每个结果是一个  regmatch_t 类型的东西：

```c
typedef struct {
    regoff_t rm_so;
    regoff_t rm_eo;
} regmatch_t;
```

两个变量存储了匹配结果首尾相对于 `string` 首地址的偏移量。

## Noise Filtering

我们希望过滤掉目标程序的噪音输出以保证父进程可以获得纯净的 strace 信息。我们有两个方向：

* 重定向目标程序的标准错误 (因为 strace 默认输出到标准错误)，但笔者并没有在 strace 的手册中查看到只重定向目标程序的选项。
* 重定向 strace 的输出：我们可以利用 strace 的 `-o` 选项来将 strace 的信息输出到一个指定的文件。事实上，我们创建的管道也是一个文件：虽然 pipe() 创建的管道是匿名的，但我们仍然可以通过 procfs 找到它：对于进程号 pid 下文件描述符 id，`/proc/pid/fd/id` 就是该文件的文件名。因此我们可以利用 `-o` 直接将 strace 的输出直接定向到管道 (甚至省掉了 dup2() 的过程)。