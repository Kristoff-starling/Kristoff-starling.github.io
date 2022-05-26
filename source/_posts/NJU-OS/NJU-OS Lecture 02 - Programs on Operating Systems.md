---
layout: post
date: 2022-02-18
title: "NJU-OS Lecture 02: Programs on Operating Systems"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Digital Logic

数字电路：

* 状态：寄存器中保存的值；
* 初始状态：RESET；
* 迁移：组合逻辑电路下计算寄存器下一个周期的值。

<!-- more -->

### Example: Seven-seg

```c
// logisim.c
#include <stdio.h>
#include <unistd.h>

#define REGS_FOREACH(_)  _(X) _(Y)
#define OUTS_FOREACH(_)  _(A) _(B) _(C) _(D) _(E) _(F) _(G)
#define RUN_LOGIC        X1 = !X && Y; \
                         Y1 = !X && !Y; \
                         A  = (!X && !Y) || (X && !Y); \
                         B  = 1; \
                         C  = (!X && !Y) || (!X && Y); \
                         D  = (!X && !Y) || (X && !Y); \
                         E  = (!X && !Y) || (X && !Y); \
                         F  = (!X && !Y); \
                         G  = (X && !Y); 

#define DEFINE(X)   static int X, X##1;
#define UPDATE(X)   X = X##1;
#define PRINT(X)    printf(#X " = %d; ", X);

int main() {
  REGS_FOREACH(DEFINE);
  OUTS_FOREACH(DEFINE);
  while (1) { // clock
    RUN_LOGIC;
    OUTS_FOREACH(PRINT);
    REGS_FOREACH(UPDATE);
    putchar('\n');
    fflush(stdout);
    sleep(1);
  }
}
```

数字电路永远是根据当前状态机的状态确定次态，循环往复。上述程序用两个寄存器 X,Y 实现了七段数码管 0 $\rightarrow$ 1 $\rightarrow$ 2 循环更新的功能 (注：这段代码使用了一些 X-macros 技巧，使得我们无需对每个寄存器写相同的代码，值得学习。)

搭配上一个简单的 python 程序，我们便可以在终端中打印这个七段数码管：

```python
# seven-seg.py
import fileinput
 
TEMPLATE = '''
\033[2J\033[1;1f
     AAAAAAAAA
    FF       BB
    FF       BB
    FF       BB
    FF       BB
     GGGGGGGGG
    EE       CC
    EE       CC
    EE       CC
    EE       CC
     DDDDDDDDD
''' 
BLOCK = {
    0: '\033[37m░\033[0m', # STFW: ANSI Escape Code
    1: '\033[31m█\033[0m',
}
VARS = 'ABCDEFG'

for v in VARS:
    globals()[v] = 0
stdin = fileinput.input()

while True:
    exec(stdin.readline())
    pic = TEMPLATE
    for v in VARS:
        pic = pic.replace(v, BLOCK[globals()[v]]) # 'A' -> BLOCK[A], ...
    print(pic)
```

> 关于该 python 程序的一些注解：
>
> * `exec(stdin.readline())` 用于将读入的内容直接当作 python 语句执行。由于之前的 C 程序在一行中用 `;` 将多个变量赋值分开，符合 python 语法，因此可以直接作为 python 程序中的变量赋值语句。
> * BLOCK[] 数组中存储了两种颜色：37 号是白色，31 号是红色。这里使用了 ANSI Escape Code，具体格式可以参考[这篇文章](https://kristoff-starling.github.io/2022/02/18/ANSI%20Escape%20Code%20-%20Quick%20Manual/)。

## Programs: Code Perspective

C 程序也是状态机：

* 状态：堆+栈；
* 初始状态：main() 的第一条语句；
* 迁移：执行一条简单语句。

从更 low-level 的角度而言，C 程序的状态机描述如下：

* 状态：全局变量+栈帧的列表 (每个栈帧里有一个 PC 表示当前执行到哪里)；

* 初始状态：`main(argc, argv)`；

* 迁移：执行 top stack frame 中 PC 处的语句，PC++。

    函数的调用本质上就是新建一个栈帧放在栈帧列表的顶部，然后跳转到该栈帧中的第一条指令；函数的返回本质上就是将顶部的栈帧 pop 掉。

### Example: Hanoi-nr

```c
// hanoi-nr.c
typedef struct {
  int pc, n;
  char from, to, via;
} Frame;

#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
#define ret()     ({ top--; })
#define goto(loc) ({ f->pc = (loc) - 1; })

void hanoi(int n, char from, char to, char via) {
  Frame stk[64], *top = stk - 1;
  call(n, from, to, via);
  for (Frame *f; (f = top) >= stk; f->pc++) {
    switch (f->pc) {
      case 0: if (f->n == 1) { printf("%c -> %c\n", f->from, f->to); goto(4); } break;
      case 1: call(f->n - 1, f->from, f->via, f->to);   break;
      case 2: call(       1, f->from, f->to,  f->via);  break;
      case 3: call(f->n - 1, f->via,  f->to,  f->from); break;
      case 4: ret();                                    break;
      default: assert(0);
    }
  }
}
```

switch-case 结构相当于给递归代码编上了“行号”。每个栈帧中的 PC 表示当前层函数执行到了第几行。

## Programs: Binary Perspective

从二进制的角度，程序仍然是状态机。

* 状态：寄存器，内存……
* 迁移：执行一条指令

如果整个程序的行为是确定的，状态机应该长成一条直线：$(M_0,R_0)\rightarrow (M_1,R_1)\rightarrow\cdots$ 。如果引入了一些 non-deterministic 的元素，比如 `rdrand` 指令 (读取一个硬件随机数到寄存器中)，那么状态机会有分叉。但无论如何，这是一个有限状态机 (内存、寄存器数量是有限的)，如果只有计算指令，有限步之后我们必然会回到一个经历过的状态。我们无法实现输入输出，甚至是停止程序。

因此，程序中有一条特殊的指令：`syscall`。程序将自己的状态机 $(M,R)$ 交给操作系统，任其修改，最终操作系统返回一个状态机 $(M',R')$，程序继续执行。因此程序=计算+syscall+计算+syscall+……

### Example: A smallest "Hello, world"

我们想象中的最小函数：

```c
void _start() {
}
```

直接运行会得到 SIGSEGV。使用 GDB 对汇编代码进行调试：

```c
0x401000 <_start>       endbr64                                            
0x401004 <_start+4>     push   %rbp                                        
0x401005 <_start+5>     mov    %rsp,%rbp                                   
0x401008 <_start+8>     nop                                                
0x401009 <_start+9>     pop    %rbp                                        
0x40100a <_start+10>    ret  
```

执行 `ret` 指令时发生错误，因此 `ret` 的本质是将 %rax 的值赋给 PC，而此时 %rax 的值是一个非法的地址。

---

用户程序只能解决“计算”的问题，因此我们甚至无法结束程序。我们必须借助系统调用才能使程序正常运行起来。

```assembly
# minimal.S
#include <sys/syscall.h>

.globl _start
_start:
  movq $SYS_write, %rax   # write(
  movq $1,         %rdi   #   fd=1,
  movq $st,        %rsi   #   buf=st,
  movq $(ed - st), %rdx   #   count=ed-st
  syscall                 # );

  movq $SYS_exit,  %rax   # exit(
  movq $1,         %rdi   #   status=1
  syscall                 # );

st:
  .ascii "\033[01;31mHello, OS World\033[0m\n"
ed:
```

前半段程序负责打印：将系统调用号 SYS_write 和输出对象，输出内容等参数放到 x86-64 制定的寄存器中，然后使用 syscall 指令；后半段负责退出。

使用 `gcc -c minimal.S && ld minimal.S && ./a.out` 命令，可以得到红色加粗的 "Hello, OS World"。

## Switching between Perspectives

记源代码为 $S$，汇编代码为 $S$，则编译器就是一个函数：$S=compile(S)$。

从状态机的视角来看，编译 (优化) 的正确性 (soundness) 的含义是：两个状态机的可观测行为完全一致。因此在保证整体语义不变的前提下，编译器可以自由地调整语句顺序，合并语句，删除无用语句等。

### Example: Compile Optimization

```c
void foo(int g)
{
    g++;
    g++;
}
```

开启 `-O1` 优化后，得到的汇编代码为

```assembly
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	c3                   	ret
```

编译器直接将 `g++` 删除了，因为没有返回，这两句话没有作用。

---

```c
extern int g;
void foo(int x)
{
    g++;
  	g++;
}
```

汇编代码为

```assembly
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	83 05 00 00 00 00 02 	addl   $0x2,0x0(%rip)        # b <foo+0xb>
   b:	c3                   	ret 
```

因为 g 是外部变量，所以这里对 g 的操作是有意义的，但连续的两次 +1 可以用一次 +2 来代替。

---
```c
extern int g;
void foo(int x)
{
    g++;
	asm volatile("nop" : : "r"(x));
  	g++;
}
```
汇编代码为

```assembly
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	90                   	nop
   5:	83 05 00 00 00 00 02 	addl   $0x2,0x0(%rip)        # c <foo+0xc>
   c:	c3                   	ret
```

这说明编译器可以调整语句的顺序，然后将两个 +1 合并成一个 +2。

但我们有手段可以让编译器无法进行这个合并：

```c
extern int g;
void foo(int x)
{
    g++;
	asm volatile ("nop" : : "r"(x) : "memory");
    g++;
}
```

汇编代码为

```assembly
0000000000000000 <foo>:
   0:	f3 0f 1e fa          	endbr64 
   4:	83 05 00 00 00 00 01 	addl   $0x1,0x0(%rip)        # b <foo+0xb>
   b:	90                   	nop
   c:	83 05 00 00 00 00 01 	addl   $0x1,0x0(%rip)        # 13 <foo+0x13>
  13:	c3                   	ret 
```

中间插入的内联汇编语句使用了 memory clobber，其意思是告诉编译器这些内联汇编代码可能会任意读取、修改内存内容，因此编译器在调整语句顺序时，不能超过这个 asm block。这句话就像一堵墙一样挡在中间，使得两处 +1 无法被合并。

该手法在操作系统的锁机制中有重要的应用 (避免编译器将 critical section 中的语句优化到外面，导致 race condition)。

## Programs: OS Perspective

操作系统眼中的程序：程序=计算+syscall+计算+syscall+…… 程序只能通过操作系统允许的方式来访问操作系统掌控的资源，从而操作系统掌握了计算机的“霸权”。

### Example: Entering a Program

> 一个 C 程序执行的第一条指令在哪里？

最合理的方法：用 gdb 打开可执行文件，然后用 `starti` 指令定位到第一条汇编指令，发现在 `/lib64/ld-linux-x86-64.so.2` 中。这是一个加载器，加载器负责跳转到 `__start()` 执行。

> **凭什么不能是 rtfm.so?**
>
> 使用 readelf 工具，可以看到在可执行文件中说明了
>
> ```
> [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
> ```
>
> 如果我们直接对 a.out 进行修改，将 ld-linux-x86-64.so.2 字段改为 rtfm.so，再创建一个从 rtfm.so 到 ld-linux-x86-64.so.2 的链接，我们就真的可以实现用我们制定的 rtfm.so 来加载程序。
>
> <u>计算机系统不存在玄学，一切都建立在确定的机制上。</u>

在执行 main() 函数之前，程序经历了很多的过程，使用了很多的系统调用，用 strace 工具可以查看这些系统调用(注：strace 会输出到标准错误，重定向时要注意)。 C 语言还支持在进入 main() 函数之前和之后做事情：

```c
// hello-goodbye.c
#include <stdio.h>

__attribute__((constructor)) void hello() {
  printf("Hello, World\n");
}

// See also: atexit(3)
__attribute__((destructor)) void goodbye() {
  printf("Goodbye, Cruel OS World!\n");
}

int main() {
}
```

> **Tip: 利用 Vim 来整理不易阅读的输入内容**
>
> strace 输出的内容杂乱难读，利用 vim 对其进行整理可以帮助我们阅读。常用的整理命令可以有：
>
> * `:%! grep [-v] word` 筛选出包含 word 的行 (-v 参数可以反向筛选)。
> * `:%s/, \r /g` 将所有的 `,` 替换为换行符。人类倾向于阅读行数多但每行都很短的内容。
> * ……

