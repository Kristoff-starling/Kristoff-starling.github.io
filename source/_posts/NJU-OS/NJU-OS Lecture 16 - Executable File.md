---
layout: post
date: 2022-04-17
title: "NJU-OS Lecture 16: Executable File"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

> 本节课默认只有静态链接。

## Overview

可执行文件的本质**描述状态机初始状态 (数据) 和迁移 (指令) 的数据结构**：可执行文件在 execve() 的时候使用，操作系统根据可执行文件的描述设置状态机并开始执行。<!-- more -->

更具体地说，可执行文件是一个描述了状态机初始状态的数据结构。状态机的初始状态一般包括以下几个方面：

* 寄存器：大部分寄存器的值由 ABI 规定 (比如哪些寄存器应该清零)，由操作系统负责设置。但也有一些重要的寄存器的值是可执行文件指定的，比如 PC 的值。
* 地址空间 (内存)：由二进制文件和 ABI 共同决定，比如某一段数据应该放到内存中的哪里，以及 argv, envp 的内容。
* 其他有用的信息 (比如调试信息)

> **什么样的文件可以被执行？**
>
> 一个文件需要满足若干条件才能被执行。假设我们有源代码 `a.c`，直接执行它 (`./a.c`) 会获得 Permission denied (bash)，即使我们用 `chmod +x a.c` 给其加上了执行权限，仍然不能正常执行。
>
> 决定一个文件是否能执行的是 execve() 系统调用。我们可以用 strace 追踪加载 `a.c` 的过程。如果 `a.c` 没有被赋予可执行权限，我们得到的是 EACCES (Permission denied)
>
> ```bash
> execve("./a.c", ["./a.c"], 0x7fff24451760 /* 66 vars */) = -1 EACCES (Permission denied)
> strace: exec: Permission denied
> +++ exited with 1 +++
> ```
>
> 如果我们给 `a.c` 赋予执行权限，得到的则是 ENOEXEC (目标文件的格式不是可识别的可执行文件格式)。
>
> ```bash
> execve("./a.c", ["./a.c"], 0x7ffe24eca680 /* 66 vars */) = -1 ENOEXEC (Exec format error)
> strace: exec: Exec format error
> +++ exited with 1 +++
> ```
>
> 这时我们再阅读 execve() 的手册的 ERROR 部分，就会有更好的理解。手册告诉我们 EACCES 类型的错误不只是因为缺少执行权限，也可能是因为对象不是正常的文件类型，比如 `strace /dev/null` 也会得到 EACCES。除了 EACCES 和 ENOEXEC 还有更多的错误类型，比如 E2BIG 表示参数/环境变量列表太长，ENOMEM 表示内核的内存空间不足等。

### Common Executable File Formats

Executable file 就是操作系统中的一个普通的对象。

Windows 中使用的是 PE 格式 (Portable Executable)。

UNIX/Linux 默认的可执行文件格式是 ELF (Executable Linkable Format)，但 UNIX/Linux 还支持 She-bang 格式的文件 (即文件开头有 `#!` 的文件)。比如我们可以在一个 `.c` 文件中书写

```python
#!/bin/python3
print('Hello')
```

给它加上执行权限后它便可以用 python 解释器运行下面的代码并输出 Hello。更神奇的是 She-bang 后面还可以跟我们自己写的程序，例如我们书写下面的两个程序：

```c
// args.c
int main (int argc, char *argv[]) {
	for (int i = 0; i < argc; i++) {
		printf("argv[%d] = %s\n", i, argv[i]);
	}
	return 0;
}
```

```
#!./args
```

第二个程序随便是什么类型 (不妨给其命名 `she-bang`)。将第一个 C 程序编译出可执行文件 args 后，给第二个文件加上可执行权限后，输入 `./she-bang` ，便可得到

```
argv[0] = ./args
argv[1] = ./she-bang
```

其实 she-bang 的原理可以理解为一个偷换参数的 execve()：我们在输入命令 `./file` 的时候，本来传给 execve() 的参数是 "./file"，但如果 file 文件的开头是 she-bang：

```
#!intepreter [optional-args]
```

那么相当于给 execve() 传入了参数 "intepreter",  "[optional-args]", "file"。这部分在 execve() 的手册中也有叙述。

> **关于 optional args 的有趣现象**
>
> 我们即使在 she-bang 后跟多个空格分开的单词，也只能传一个参数。这是一个 UNIX 的历史遗留问题。比如我们将刚才的 she-bang 修改为
>
> ```
> #!./args Hello OS World
> ```
>
> 则输出结果为
>
> ```
> argv[0] = ./args
> argv[1] = Hello OS World
> argv[2] = ./she-bang
> ```

> **为什么我使用 `strace ./she-bang` 观测，但没有看到显式的 “偷梁换柱” 现象？**
>
> 在 linux kernel 的源码中，execve() 会调用 `do_open_execat("./she-bang")`，里面有一个函数是 loadelf，有一个函数是 loadscript，在 load_script() 中，内核有权限打开一个文件并检查器开头是不是 `#!`，如果是就会把后面的解释器路径读出来，然后递归地调用 `do_open_execat("./args")`。因此这个“偷梁换柱”的过程是在 execve() 内部完成的。

## Parsing Executable File

GNU binutils 提供了一系列与二进制文件打交道的工具。这些工具的本质都是查看/修改数据结构中的内容。

### Debugging Information

```c
// segfault.c
#include <stddef.h>

void bar() {
  *(int *)NULL = 1;
}

void foo() {
  bar();
}

int main() {
  foo();
}
```

上述程序对一个空指针解引用，显然会发生段错误。如果我们用 GDB 调试，我们可以抓到出错的行。使用 `backtrace` / `where` 还可以打印出完整的函数调用序列，以及每个函数对应到源代码的行数：

```
(gdb) where
#0  bar () at segfault.c:4
#1  0x0000555555555151 in foo () at segfault.c:8
#2  0x0000555555555166 in main () at segfault.c:12
```

我们还可以感受 addr2line 工具的威力：在可执行文件 segfault 中找到 foo() 函数对应的地址 ADDR，然后输入

```bash
addr2line ADDR segfault
```

便可以得到该地址对应到源代码 `segfault.c` 中的行号：

```
~/os-demo/Lecture16/segfault.c:7
```

我们针对二进制文件操作，却可以得到源代码相关的信息，这是因为我们的二进制文件中保留了调试信息 (当然，如果我们编译的时候不带 `-g` 选项，使用 GDB backtrace 的时候就无法定位到源文件行号)。如果我们用 `readelf -S file` 查看可执行文件 file 的 section header table，可以看到有 debug info 等节，这就是调试信息。

现在调试信息采用的标准是 DWARF debugging standard。调试信息可以理解为将 assembly (机器状态) 映射到 "C语言世界" 状态的函数。我们 C 语言的形式语义是状态机之间的转移 (high level 的状态机，比如栈帧，行号等)，编译器的工作是将其翻译成汇编指令，汇编指令描述了更底层的状态机的转移 (memory, register etc.)，调试信息的作用就是将一个底层的状态机状态映射到 C 语言世界的一个状态机状态。

> **编译器的“摆烂”**
>
> 将机器状态映射到 C 语言状态是一件很困难的事情。比如我们可能有函数內联，这让 backtrace 变得困难；再比如比较激进的编译优化会在语义不变的前提下改写汇编程序而不是逐句翻译，这使得有些汇编状态其实根本找不到对应的 C 语言状态。因此，我们在调试时经常碰到各种各样不正确的调试信息，以及一些明明可以打印，却被摆烂的 `<optimized out>`。
>
> ```c
> // popcount.c
> #include <stdio.h>
> 
> __attribute__((noinline))
> int popcount(int x) {
> int s = 0;
> int b0 = (x >> 0) & 1;
> s += b0;
> int b1 = (x >> 1) & 1;
> s += b1;
> int b2 = (x >> 2) & 1;
> s += b2;
> int b3 = (x >> 3) & 1;
> s += b3;
> return s;
> }
> 
> int main() {
> printf("%d\n", popcount(0b1101));
> }
> ```
>
> `popcount.c` 可以正确地打印出一个数二进制表示中有多少个 1。但如果我们用 GDB 调试，在不开 O2 的情况下，我们刚进入 popcount() 时 `p x` 会发现 GDB 输出了不正确的值 0。在开 O2 的情况下，我们运行到 popcount() 的 ret 时，会发现变量 s 被 optimized out 了，但 %eax 的值 3 是正确的……这都是错误或不得当的调试信息。

### Example: Stack Unwinding

这些功能其实并不神秘，利用 x86 架构的栈帧结构，我们也可以实现一个简易的 stack unwinding：

```c
// unwind.c
#include <stdio.h>
#include <stdlib.h>

const char *binary;

struct frame {
  struct frame *next; // push %rbp
  void *addr;         // call f (pushed retaddr)
};

void backtrace() {
  struct frame *f;
  char cmd[1024];
  extern char end;

  asm volatile ("movq %%rbp, %0" : "=g"(f));
  for (; f->addr < (void *)&end; f = f->next) {
    printf("%016lx  ", (long)f->addr); fflush(stdout);
    sprintf(cmd, "addr2line -e %s %p", binary, f->addr);
    system(cmd);
  }
}

void bar() {
  backtrace();
}

void foo() {
  bar();
}

int main(int argc, char *argv[]) {
  binary = argv[0];
  foo();
}
```

运行代码 (不开优化)，我们确实可以得到一个完整的函数调用链：

```
000000000040199e  /home/starling/os-demo/Lecture16/unwind.c:26
00000000004019b3  /home/starling/os-demo/Lecture16/unwind.c:30
00000000004019e1  /home/starling/os-demo/Lecture16/unwind.c:34
00000000004039b2  ??:?
```

backtrace() 函数的实现原理很简单：我们默认该程序运行在 x86 架构上，x86 架构在调用一个函数时总是使用 call 指令，call 指令会向栈中写入返回地址，然后 PC 跳转到目标函数的第一条指令。目标函数的前两条指令一定是

```assembly
push %rbp
mov %rsp, %rbp
```

执行完了以后，栈上的内容为返回地址和旧 rbp 值，现在的 rbp 寄存器指向存储旧 rbp 值的地址。因此我们只要不断向前寻找 rbp 值，就可以实现栈帧的回溯，每次找 rbp 值上一个 8 字节区域的内容，就能找到返回地址，利用 addr2line 对这个返回地址定位，就能找到调用当前函数的 caller。

上述示例代码使用一个 `struct frame` 结构比较精巧地完成了这件事，它刚开始用一句內联汇编把 %rbp 的值放到了 f 中，f 的结构为

```c
struct frame {
    struct frame *next;
    void *addr;
}
```

这样 next 存储的正好是旧 rbp，addr 存储的正好是当前函数的返回地址 (栈是从上往下生长的)。`end` 变量记录了代码段的结束位置，回溯到 end 之上就可以结束了。

> **GDB 的强大**
>
> 如果我们使用 O2 优化，示例代码将不能正常工作，因为这些函数都被內联了，我们只能看到一层调用。但如果用 GDB 的 backtrace，我们仍然可以看到完整的函数调用链——这个调用过程是 GDB 虚构出来的正确信息。

## Linking and Relocation

```c
// hello.c
void hello() {}
```

```c
// main.c
void hello();

int f(int a, int b) {return a + b;}

int main () {
    hello();
}
```

上述两个文件分别编译成可重定位文件后，最后链接在一起就可以执行。我们关心 `main.c` 到底是如何找到外部的 hello() 函数的地址的。如果我们查看 `main.o` 的代码段信息：

```assembly
0000000000000000 <f>:
   0:   f3 0f 1e fa             endbr64 
   4:   8d 04 37                lea    (%rdi,%rsi,1),%eax
   7:   c3                      ret

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64 
   4:   48 83 ec 08             sub    $0x8,%rsp
   8:   31 c0                   xor    %eax,%eax
   a:   e8 00 00 00 00          call   f <main+0xf>
   f:   31 c0                   xor    %eax,%eax
  11:   48 83 c4 08             add    $0x8,%rsp
  15:   c3                      ret
```

会看到像 f() 这样完全确定的函数编译器已经完全生成好了代码，编译器还没有填的是 call 后面的地址偏移，因为链接之前它不知道 hello() 在哪里，因此只能暂时填 0 摆烂。

链接结束后 main() 在 0xb 处填写的 offset 应该满足下面的 assertion：

```c
// hello.c
#include <stdio.h>
#include <stdint.h>
#include <assert.h>

void main();
void hello() {
    void *p = (void *)main + 0xa + 1;
    int32_t offset = *((int32_t *)p);
    assert((char *)main + 0xf + offset == (char *)hello);
}
```

(注：开始跳转的地址是 `main +0xf` 是因为 x86 的 call 指令是从下一条指令的地址开始跳转的。)

为了满足这个 assertion，我们容易计算出要填写的 offset 应该满足 $S+A-P$ 的格式，其中 $S$ 是目标函数的地址 (在这里是 `(void *)hello`)，$A$ 是一个偏移量，(在这里是 $-4$)，$P$ 是下一条指令的地址 (在这里是 `(void *)main + 0xb`)。如果我们用 readelf 查看 `main.o` 的重定位表，可以看到

```
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000b  000c00000004 R_X86_64_PLT32    0000000000000000 hello - 4
```

这里的  "Offset" 就是 $P$，"Addend" 就是 $A$，"Sym" 就是 $S$。

因此，通俗来说，

* 编译器 (gcc) 就是将 high-level semantics (C 语言) 转换成 low-level semantics (汇编)。
* 汇编器 (as) 就是将 low-level semantics 转换成 binary semantics (状态机容器)，这个部分几乎是一一对应地翻译，对于暂时没法填的要留下重定位信息，重定位信息本质上就是对填写内容的 assertion。
* 链接器 (ld) 负责合并所有容器，得到一个完整的状态机，除了我们指定的 `.o` 文件外，链接器还会将一些 C Runtime Objects 链接进来。找不到符号/重复强符号的错误也是在链接阶段报出。

在这种理解下，我们很容易设计一个自己的“简易二进制文件格式“：

```c
struct executable {
    uint32_t entry;
    struct segment *segments;
    struct reloc *relocs;
    struct symbol *symbols;
};
struct segment {uint32_t flags, size; char data[0];}
struct reloc {uint32_t S, A, P; const char *name;}
struct symbol {uint32_t offset, const char *name;}
```

随着各种需求的加入，我们就会慢慢理解 ELF 中各种信息设置的含义。

