---
title: "Lecture 18: Xv6 Code Guide"
linktitle: '18: xv6 概览'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 18
---
> Xv6 中的 `.d` `.sym` `.asm` 文件是什么？

`.d` 文件描述了 Makefile 中文件所需的依赖关系。我们考虑如下程序：

```makefile
a.o: a.c
	gcc -c a.c
```

```c
// a.c
#include "a.h"
```

```c
// a.h
// Cannot compile
```

Makefile 的逻辑是：`a.o` 依赖文件 `a.c`，在执行 make 时，如果 `a.c` 相较于上一次 make 发生过改动，则会再次执行编译指令，如果没有改动则不执行。但当我们的 `a.c` include 了 `a.h` 之后，实际上 `a.o` 多了一个间接的依赖关系：当 `a.h` 发生改动时，我们也应该重新执行编译指令。但很遗憾，除非在 Makefile 中添加 `a.h` 依赖，否则 Makefile 不会自己做到这一点。

大型项目中一个 C 程序可能有很多很多的 include。如果将这些 include 全部填到 Makefile 中，Makefile 将变得难以维护。因此我们利用编译器生成了 `.d` 文件，`.d` 文件中包含了一个程序依赖的头文件列表，且这个列表是 Makefile 可以直接解析的，这样我们解决了依赖问题。

## Xv6 Framework

`/kernel` 是内核代码，Makefile 会将所有的源代码文件编译后根据 `/kernel/kernel.ld` 的链接脚本链接生成一个 `kernel` 可执行文件。`/user` 是用户程序代码，Makefile 会给将每个形如 `name.c` 的用户程序编译生成一个 `_name` 可执行文件。`/mkfs` 是生成文件系统的代码，`/user` 中所有下划线开头的可执行文件会被放入文件系统。

> **工具的配置**
>
> 好的工具会提升阅读代码的体验，使用 vscode 是一个很好的选择。为 vscode 添加正确的 `compile_command.json` 可以使 vscode 正确地跳转。`compile_command.json` 的原理是提供每个文件依赖的 include path。我们可以用 bear 工具直接爬 `make qemu` 的信息，它会自动生成 `.json` 文件。

## Code: xv6 System Call

因为启动第一个进程的时候 xv6 还没有初始化文件系统，无法通过 exec() 系统调用来加载 image，所以 xv6 的第一个进程 initcode 是写死的一段汇编代码 (在 forkret() 中有初始化文件系统的相关代码)，具体的信息可以见 MIT 6.S081 Lecture 03。

通过 `ecall` 指令走 trampoline 然后进入 usertrap() 的流程可以见 MIT 6.S081 Lecture 6。

> **为什么我在 GDB 中加断点 `b *0x0` 会报错："cannnot access memory?"**
>
> 2020版之前的 xv6 需要在 `.gdbinit.tmpl-riscv` 中添加一条配置：
>
> ```bash
> set riscv use-compressed-breakpoints yes
> ```

