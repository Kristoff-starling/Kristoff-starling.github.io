---
title: "Lecture 14: C Library"
linktitle: '14: LibC'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 14
---

`sh-xv6.c` 可以在 freestanding 的环境下直接运行：我们最终使用 `ld sh-xv6.o -o sh` 生成二进制文件，这意味着二进制文件中有且仅有 `sh-xv6.o` 中的函数。

我们平时在编写程序的时候显然不希望在 freestanding 的环境下编程——联想我们在 `sh-xv6.c` 中，读取字符串都要用內联汇编的系统调用，这太糟糕了。我们自然而然地希望有一些封装好的库函数可以使用。

## Libc: Overview

### Portability

在 freestanding 环境下我们也有一些库可以使用。如果我们想写出可移植性强的代码，我们应该使用 libc 中提供的数据类型：

* 比如我们要保存一个指针的值，瞎猜一个 `int` `long` 不是好的选择，我们应该用 `intptr_t` (该类型在 `stdint.h` 中定义)；比如我们想保存一个 4 字节的数据，应该使用 `int32_t`。(`long` 的长度是随架构变化的) 等等。

* C 语言支持边长参数。例如 printf() 的声明是：

    ```c
    int printf(const char *format, ...);
    ```

    在 x86 架构中，参数是一个接着一个放在栈上的，因此在一些 i386 的代码中我们可能会看见下面的这种写法：

    ```c
    void **p = (void *)&fmt;
    // use p[i] to get the address of the ith argument
    ```

    但 riscv64, x86-64 等架构是用寄存器传递参数的。为了写出可移植性更好的代码，我们应该使用 `va_list` 类型，然后用 `va_start()` `va_arg()` `va_end()` 等 API 去访问各个参数 (该类型在 `stdarg.h` 中定义)。

    > **使用 libc 一定高效吗?**
    >
    > 不一定。GCC 的遗留问题使得使用 libc 编译出的汇编代码可能会“很笨”。例如 `sh-xv6.c` 中关于系统调用的这段代码：
    >
    > ```c
    > long syscall(int num, ...) {
    > va_list ap;
    > va_start(ap, num);
    > register long a0 asm ("rax") = num;
    > register long a1 asm ("rdi") = va_arg(ap, long);
    > register long a2 asm ("rsi") = va_arg(ap, long);
    > register long a3 asm ("rdx") = va_arg(ap, long);
    > register long a4 asm ("r10") = va_arg(ap, long);
    > va_end(ap);
    > asm volatile("syscall"
    >  : "+r"(a0) : "r"(a1), "r"(a2), "r"(a3), "r"(a4)
    >  : "memory", "rcx", "r8", "r9", "r11");
    > return a0;
    > }
    > ```
    >
    > 如果我们分别为 1,2,3,4 个参数的系统调用写 4 个接口，像下面这样：
    >
    > ```c
    > long syscall4(long num, long a1, long a2, long a3, long a4) {
    >  long a0 = num;
    >  asm volatile("syscall"
    >      : "+r"(a0) : "r"(a1), "r"(a2), "r"(a3), "r"(a4)
    >      : "memory", "rcx", "r8", "r9", "r11");
    >  return a0;
    > }
    > ```
    >
    > 观察两者在 -O2 优化下反汇编的结果，可以看到前者在寄存器和栈上来回倒腾数据，而后者只用了很少的代码就完成了工作。

* printf() 的模式串中也涉及可移植性的问题。比如我们可能常常这么写程序：

    ```c
    int64_t a = 1;
    printf("%ld\n", a);
    ```

    但 `%ld` 是用于打印一个 `long` 长度的变量的，`long` 的长度随架构变化而变化，上述代码在 32 位的架构上就会报 warning。`inttypes.h` 其实为模式串的可移植性也设计了一套符号 (但很复杂)。

### Convenience

并不是所有的系统调用都像 fork() 那样简单易用，因此库函数有义务将系统调用封装成更易用的样子，比如：

```c
extern char **environ;
char *argv[] = {"echo", "hello", "world", NULL, };
if (execve(argv[0], argc, environ) < 0) perror("exec");
```

运行上述程序，我们会得到 `exec: No such file or directory`。这是因为 execve() 要求第一个参数必须是完整的 pathname。一些高情商的 API 是这样封装的：

```c
execlp("echo", "echo", "hello", "world", NULL);
system("echo hello world");
```

exec() 家族的库函数有很多：

* l 系列的函数支持变长参数，我们不用单独开一个参数数组 argv[] 而可以直接把要传的参数放进去。与之相对地，v 系列的函数将参数保存好以后，将 argv[] 放到函数中即可，这对于多次使用同样参数很有利。
* p 系列的参数在第一个 pathname 中没有 `/` 的情况下，会遍历 PATH 环境变量中的所有路径，将其接在文件名前面并尝试 execve()。
* e 系列可以不使用系统环境默认的环境变量，而是传入一个 envp[] 数组指定环境变量 (非 e 系列的函数默认以 `extern char *environ[]` 的内容作为环境变量)。

更多的内容可以查看手册。

## Encapsulation: Calculation

纯计算的工作是我们大量使用并想要封装的，比如统计字符串的长度、给一段内存赋值等等。这看上去是非常简单的事情，但如果考虑到效率，安全性等因素，事情就变得复杂了。比如如下一段 memset 的实现：

```c
vodi *memset(void *s, int c, size_t n) {
    for (size_t i = 0; i < n; i++)
        ((char *)s)[i] = c;
    return s;
}
```

如果有多个线程调用该程序，它能保证安全性吗？数据会互相覆盖吗？我们可以先用 libc 标准的 memset() 来测试一下多线程下的表现：

```c
// memset-race.c#include "thread.h"char buf[1 << 30];void foo(int id) {  memset(buf, '0' + id, sizeof(buf) - 1);}int main() {  for (int i = 0; i < 4; i++)    create(foo);  join();  puts(buf);}
```

多个线程同时对一段内存赋值，这是一个标准的数据竞争。运行这段程序我们可以看到 1,2,3,4 都有被输出。根据标准，标准库只对“标准库内部数据”的线程安全性负责 (比如 printf 的 buffer 是有锁保护的)，对外部数据库函数没有义务处理数据竞争。

除此之外，封装的“好用”也是我们的一大目标。我们可以对比 C 语言的排序函数和 C++ 的排序函数：

```c
void qsort(void *base, size_t nmemb, size_t size,                  int (*compar)(const void *, const void *));
```

```c++
sort(xs.begin(), xs.end(), [](auto &a, auto *b) {...});
```

可以看到 C++ 的库函数明显更好用：我们可以直接用迭代器的 begin 和 end，还可以用 lambda 表达式把比较函数直接写在 sort() 里面。

## Encapsulation: File Descriptors

操作系统有义务封装对象 (比如终端，管道，文件 etc.) 并暴露合适的 API 给应用程序。由于 UNIX 秉持 everything is a file 的哲学，所以封装对象的核心就是文件描述符。

考虑如下程序：

```c
int main () {    FILE *fp = fopen("a.txt", "w");    fprintf(fp, "Hello, world\n");    return 0;}
```

它会向 `a.txt` 输出 Hello, world。这里的文件指针事实上指向的就是文件结构体。我们可以尝试用 GDB 调试这段代码，用 `p *fp` 打印 fp 指针指向的内容：

```
{_flags = -72539004, _IO_read_ptr = 0x0, _IO_read_end = 0x0,   _IO_read_base = 0x0, _IO_write_base = 0x0, _IO_write_ptr = 0x0,   _IO_write_end = 0x0, _IO_buf_base = 0x0, _IO_buf_end = 0x0,   _IO_save_base = 0x0, _IO_backup_base = 0x0, _IO_save_end = 0x0,   _markers = 0x0, _chain = 0x7ffff7fa15e0 <_IO_2_1_stderr_>, _fileno = 3,   _flags2 = 0, _old_offset = 0, _cur_column = 0, _vtable_offset = 0 '\000',   _shortbuf = "", _lock = 0x555555559380, _offset = -1, _codecvt = 0x0,   _wide_data = 0x555555559390, _freeres_list = 0x0, _freeres_buf = 0x0,   __pad5 = 0, _mode = 0, _unused2 = '\000' <repeats 19 times>}
```

可以看到这是一个文件描述符为 3 的文件。如果我们用 `p *stdin/*stdout/*stderr` 打印，也可以看到类似的内容，标准输入/输出/错误都是文件。

另一个有意思的 API 是 popen()，它可以打开一个管道。这其实是一个设计的有缺陷的 API：手册中明确提到 UNIX pipe 是单向的：它有一个读口和一个写口，而文件指针 `FILE *` 只能封装一个文件描述符，所以使用 popen() 必须指定从管道读还是写向管道。(现代的编程语言对管道 API 封装的更好。)

## Encapsulation: Utilities

### err

> 为什么 `cat nonexist.c` `gcc nonexist.c` 等命令输出的都是 "xxx: nonexist.c: No such file or directory"?

我们也可以用库函数山寨一个类似的版本：

```c
// err-message.c#include <err.h>#include <stdio.h>int main (){	const char *filename = "nonexist.c";	FILE *fp = fopen(filename, "r");	if (!fp) warn("%s", filename);	return 0;}
```

`err.h` `errno.h` 提供若干这样将错误信息输出到标准错误的函数，好用的函数还有 perror()，err() 等。err() 可以在输出错误信息后直接退出程序，如：

```c
fd = open(filename, O_RONLY, 0);if (fd == -1)    err(EXIT_FAILURE, "%s", filename);
```

这些函数的原理是根据全局变量 errno 的数值，打印上一次错误的原因。手册中对于 errno 的解释为：

> errno  is  defined  by  the ISO C standard to be a modifiable lvalue of type int, and must not be explicitly declared; errno may  be  a  macro. errno  is  thread-local;  setting  it in one thread does not affect its value in any other thread.

值得注意的一点是 errno 是线程独享的 thread-local 变量。从这里我们可以看到协程相对于线程更轻量级体现在何处。

### environ

```c
// env.c#include <stdio.h>int main() {  extern char **environ;  for (char **env = environ; *env; env++) {    printf("%s\n", *env);  }}
```

C 库函数为我们提供了一个指针 environ，通过它我们可以找到环境变量列表。一个自然的问题是：这个指针是什么时候指向正确的位置的？

我们可能认为在 CPU reset 到 boot 完成期间 environ 就被设置好了，但考虑到一个程序的环境变量是在 execve() 的时候传进去的，不同程序可以不一样，这个变量显然应该是在进程加载启动时才设置的。

我们可以用 GDB 观测这个行为。在静态加载下，用 `gdb ./env` 启动上述程序并用 `starti` 停在第一条汇编指令，查看 `p (char **)environ` ，发现此时 environ 还是空指针。我们可以在 environ 上加监视点 `wa (char **)environ` ，即可看到在 `__libc_start_main()` 函数中修改了 environ。

> 为什么我动态链接时无法查看 environ 的信息？

## Encapsulation: Address Space

libc 为我们封装好了地址空间。我们可以通过 mmaps() 修改地址空间映射，也可以通过 malloc() 和 free() 从堆区里申请/释放空间。

想要优化 malloc()/free() 的性能，我们要对 workload 有正确的假设。指导思想：$O(n)$ 大小的对象分配后至少有 $\Omega(n)$ 的读写操作，否则就是 performance bug。

* 越小的对象创建/分配越频繁 (e.g. 字符串，临时对象等，生存周期不长)
* 较为频繁地分配中等大小对象 (e.g. 复杂的对象，较大的数组)
* 低频率的大对象 (e.g. 巨大的容器，分配器；很长的生存周期)

核心优化思想：Fast path + slow path，争取在 fast path 上做到 $O(1)$ (无锁或无 contention)。计算机系统的世界中处处是 fast path + slow path，比如 memory hierarchy。