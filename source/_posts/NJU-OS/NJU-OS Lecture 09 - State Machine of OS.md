---
layout: post
date: 2022-03-17
title: "NJU-OS Lecture 09: State Machine of OS"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Bare-metal & Software

数字电路本身就是一个巨大的状态机。硬件厂商会保证 CPU reset 后处理器处在一个确定的状态，各个单元的固定的值会写在手册中。以 x86 为例，手册规定了 CPU reset 后 rip 的值会是 0xfff0，EFLAGS 的值为 0x2，当前硬件运行在 16-bit 模式中，不响应中断。

<!-- more -->

CPU 只是一个周而复始的取值译码执行的东西，因此它马上会做的事情就是取出 0xfff0 处的指令并执行。0xfff0 处通常是一个跳转指令，PC 会跳转到固件代码执行。固件 (Firmware) 是硬件厂商写死在一块 ROM 上的代码，常见的固件程序有 BIOS 和 UEFI 两种，后者更加现代机制更加复杂，这里简介 Legacy BIOS 的过程：

BIOS 的主要工作是扫描各个硬盘/U盘/软盘……然后将第一个可引导的设备的第一个扇区 (512B) 加载到地址 0x7c00 处。可引导的设备的第一个扇区会遵循主引导记录 (Master Boot Record, MBR) 的格式。如果我们使用命令

```c
cat executable-x86_64-qemu | head -c 512 | xxd
```

查看一个镜像文件的前 512 个字节，我们可以看到其末尾一定是 0x55aa。这是 MBR 的结束标志。

此时硬件仍然工作在 16-bit 模式上。规定 `CS:IP=0x7c00` ，即 `(R[CS]<<4) | R[IP] == 0x7c00` ，其他没有任何约束。Firmware 执行结束后，PC 跳转到 0x7c00 执行，刚刚加载过来的是 Legacy boot (boot loader)，它负责完成操作系统的加载：初始化栈和堆区，为 main() 传递参数等等。

## Code: Firmware

QEMU 是一个全系统模拟器，我们可以通过 GDB 远程连接的方式来调试 QEMU，从而观测各个时刻的系统状态。

一个启动 QEMU 并使用 GDB 监听的脚本如下：

```bash
#!/bin/bash

qemu-system-x86_64 \
  -smp 1\
  -S -s\
  -drive format=raw,file=amgame-x86_64-qemu &
pid=$!

gdb -x remote.gdb ; kill -9 $!
```

`qemu-system-x86_64` 启动了一个 x86-64 架构的 QEMU 模拟器，之后的各个选项和参数的意义如下：

* `-smp 1` 指定了 QEMU 模拟单核 CPU。
* `-S` 选项表示让 QEMU 停止在 CPU reset 后的第一条指令上。
* `-s` 选项是 `-gdb tcp::1234` 的简写，表示在端口 1234 打开一个 gdbserver。
* `-drive format=raw,file=amgame-x86_64-qemu` 告诉了 QEMU 用于引导的设备。这里笔者使用了 Lab0 中 amgame 的镜像。

gdb 的 `-x` 选项后面可以跟一个脚本，表示在启动 gdb 之后自动运行后面的脚本。笔者在脚本中只做了一行配置：

```bash
target remote localhost:1234
```

表示连接到 gdbserver。之前我们给运行 QEMU 的进程分配了一个进程号，现在用 `kill -9 $!` 可以在退出 gdb 之后帮我们自动杀死 QEMU 的进程。

运行这个脚本，我们可以看到 QEMU 停在了 CPU reset 后的初始状态。使用 `i r` 命令打印寄存器信息，可以看到 `rip = 0xfff0`，`eflags = 0x2` 等，符合手册的约定。此时如果使用命令 `x/16xb 0x7c00` 查看地址 0x7c00 附近的内存内容，可以看到全部是 0 (注：`x` 是打印，`16` 是打印 16 个 byte，`x` 是按照十六进制打印，`b` 是将各个 byte 分开)。

```assembly
0x7c00:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x7c08:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

想要监控 BIOS 的行为，我们可以用命令 `wa *0x7c00` 在这个地址上打一个断点，然后 continue，该地址的值出现变化时 GDB 会停下并通知我们。

当前系统仍处于 16 位模式，因此需要用 cs 和 rip 两个寄存器来定位指令的位置。使用 `x/i ($cs * 16 + $rip)` 查看当前指令：

```assembly
0xfa591:	rep insl (%dx),%es:(%edi)
```

这是一条内存拷贝指令，刚开始可以看到 `%es:(%edi) == 0x7c00` ，随着 repeat 的推进，`(%edi)` 的值逐渐变大。如果再查看 0x7c00 附近的内容，可以看到 MBR 正在被逐渐搬入：

```assembly
0x7c00:	0xfa	0x31	0xc0	0x8e	0xd8	0x8e	0xc0	0x8e
0x7c08:	0xd0	0xb8	0x01	0x4f	0xb9	0x12	0x01	0xbf
0x7c10:	0x00	0x40	0xcd	0x10	0x00	0x00	0x00	0x00
0x7c18:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
```

## State Machine of OS

Firmware 和 boot loader 共同完成了操作系统的加载。进入 C 代码后，操作系统将完全遵循 C 语言的形式语义（当然，调用 AbstractMachine API 的部分是对 C 语言形式语义的补充）。

AbstractMachine 对 C 程序语义的扩展主要是以下几个部分：

* TRM + MPE
    * 完全等同于多线程。将整个系统的状态机从一条链变成多条链，之后的执行将是 non-deterministic 的。
    * IOE API 相当于库函数。
* CTE
    * 允许创建多个执行流。
    * yield() 会主动切换，中断会被动切换。
* VME
    * 创建一个“经过地址翻译的执行模式”。

整个系统的状态机大致可以看成如下过程：

```mermaid
flowchart LR
	subgraph CPUReset
		PC=0xfff0 --- EFLAGS=0x2 --- CR0=... --- S0[...]
	end
	id0((init))-. CPU Reset .-> CPUReset
	CPUReset -. Firmware .-> Boot
	subgraph Boot
		PC=0x7c00 --- MBR[MBR at 0x7c00] --- ...
	end
	Boot -. OS booter .-> main
	subgraph main
		g[global] -.- sf[stack frame] -.- PC -.- S[...]
	end
	main -. mpe_init .-> multithread
	subgraph multithread
		gg[global]
		sf1[stack frame] --- pc1[PC] --- o1[...]
		sf2[stack frame] --- pc2[PC] --- o2[...]
	end
	multithread -. CPU1 .-> state1((...))
	multithread -. CPU2 .-> state2((...))
	multithread -. Interrupt .-> state3((...))
```

## Code: AbstractMachine

> **Unix Philosophy**
>
> Makefile 中的变量的定义可能散落在世界各处，因此静态地读 Makefile 效果可能不好。好的方式是直接观测 Makefile 做了怎样的事情。这里提供一套命令：
>
> ```bash
> make mainargs=HelloOS run -nB \
> | grep -ve '^\(\#\|echo\|mkdir\|make\)' \
> | sed "s#$AM_HOME#\$AM_HOME#g" \
> | sed "s#$PWD#.#g" \
> | vim -
> ```
>
> * make 的 `-n` 选项可以只输出执行的命令而不真正执行，`-B` 选项可以忽略已经编译好的内容从头编译。
> * grep 用于筛选，`-v` 选项表示反向筛选，`-e` 表示使用正则表达式匹配。
> * sed 可以作替换，这里将冗长的路径名替换成了短的。
> * vim - 命令可以将输出导入到 vim 中查看。
>
> 在 vim 中使用 `:%s/ /\r  /g` 可以将空格改为换行，从而提高可读性。

通过观察 Makefile 的输出结果，我们可以看到它是如何创建镜像文件以及传输命令行参数的：

```bash
( cat $AM_HOME/am/src/x86/qemu/boot/bootblock.o;
  head -c 1024 /dev/zero; 
  cat ./build/threados-x86_64-qemu.elf )
> ./build/threados-x86_64-qemu
```

`bootblock.o` 是主引导记录。`head -c 1024 /dev/zero` 取出了 1024B 的 \0。`threados-x86_64-qemu.elf` 是我们之前编译、链接得到的 elf 文件。将这三者拼接起来输出到文件 `threados-x86_64-qemu` 中，就制作好了镜像文件。

```bash
( echo -n HelloOS; ) | dd if=/dev/stdin of=./build/threados-x86_64-qemu 
 bs=512 count=2 seek=1 conv=notrunc status=none
```

参数 HelloOS 被输出后，dd 指定 input file 为 stdin，output file 为我们刚刚制作的镜像，将参数写到了第二个扇区里 (即 MBR 后面紧跟着的扇区)。

之前我们用 GDB 调试了固件将 MBR 加载到 0x7c00 的过程，现在我们可以继续用 GDB 调试 OS booter 的运行过程。

> **没有 OS booter 的符号表怎么办？**
>
> 仔细阅读 AbstractMachine 的 Makefile，我们发现 `bootblock.o` 就是主引导记录，它的生成过程如下：
>
> * `gcc -static -m32 -fno-pic -g -Os -nostdlib -Ttext 0x7c00 -I$AM_HOME/am/src -o bootblock.o start.S main.c`，该命令编译 `am/src/x86/qemu/boot/` 目录下的 `main.c` 和 `start.S` 并链接成一个 `bootblock.o`。
>
>     为了生成带有调试信息的二进制文件，我们应当加入 `-g` 选项。
>
> * `python genboot.py bootblock.o` 对 `bootblock.o` 做了进一步的修改，阅读 `genboot.py` ：
>
>     ```python
>     import os, sys, pathlib, subprocess
>         
>     f = pathlib.Path(sys.argv[1])
>     try:
>         objcopy = os.getenv('CROSS_COMPILE', '') + 'objcopy'
>         data = subprocess.run(
>             [objcopy, '-S', '-O', 'binary', '-j', '.text', f, '/dev/stdout'],
>             capture_output=True).stdout
>         assert len(data) <= 510
>         data += b'\0' * (510 - len(data)) + b'\x55\xaa'
>         f.write_bytes(data)
>     except:
>         f.unlink()
>         raise
>     ```
>
>     它的主要工作是用 objcopy 工具将 `bootblock.o` 中的关键内容拷贝出来，`-S` 选项表示丢弃重定位信息，符号信息，调试信息等不需要的内容；`-j .text` 表示只保留代码节。拷贝出来后在字节流的后面补上若干 0，最后加上 0x55aa，以形成一个 MBR。
>
> 为了获得符号表，笔者做了如下修改：将第一步生成的 `bootblock.o` 备份一份重命名为 ` boot-sym.o` 用于加载符号表，并在 gdb 启动时的自动化脚本中添加：
>
> ```bash
> symbol-file /path/to/boot-sym.o
> ```

在 GDB 中打断点 `wa ($rip != 0x7c00)` (或者 `b _start`) 即可定位到 OS booter。此时机器仍然处于 16 位模式，GDB 不支持 16 位的调试，打印出来的汇编命令会出现错误。我们可以在 `$AM_HOME/am/src/x86/qemu/boot/start.S` 中找到对应的指令。可以看到在执行了不多的几条指令后，PC 跳转到了函数 start32。start32 执行了几条简单的指令后跳转到了 load_kernel()。(在 load_kernel() 中终于可以用 layout src 调试了)

load_kernel() 和 nanos-lite 中的 loader() 功能类似：将 elf 文件中的各个段拷贝到内存的指定地址 0x8000，并跳转到入口函数启动内核。

```c
Elf32_Ehdr *elf32 = (void *)0x8000;
Elf64_Ehdr *elf64 = (void *)0x8000;
int is_ap = boot_record()->is_ap;

if (!is_ap) {
    // load argument (string) to memory
    copy_from_disk((void *)MAINARG_ADDR, 1024, -1024);
    // load elf header to memory
    copy_from_disk(elf32, 4096, 0);
    if (elf32->e_machine == EM_X86_64) {
      load_elf64(elf64);
    } else {
      load_elf32(elf32);
    }
}
```

这段代码调用 copy_from_disk() 将 mainargs 以及 elf 文件从 disk 中拷贝出来，再调用 load_elf() 将其加载到 0x8000 位置。在执行 if 之前通过 `p *elf64` 查看 0x8000 附近的内容，可以看到各个字段全部为空。

```bash
(gdb) p *elf64
$1 = {e_ident = '\000' <repeats 15 times>, e_type = 0, e_machine = 0,
  e_version = 0, e_entry = 0, e_phoff = 0, e_shoff = 0, e_flags = 0,
  e_ehsize = 0, e_phentsize = 0, e_phnum = 0, e_shentsize = 0, e_shnum = 0,
  e_shstrndx = 0}
```

```c
static void load_program(uint32_t filesz, uint32_t memsz, uint32_t paddr, uint32_t offset) {
  copy_from_disk((void *)paddr, filesz, offset);
  char *bss = (void *)(paddr + filesz);
  for (uint32_t i = filesz; i != memsz; i++) {
    *bss++ = 0;
  }
}

static void load_elf64(Elf64_Ehdr *elf) {
  Elf64_Phdr *ph = (Elf64_Phdr *)((char *)elf + elf->e_phoff);
  for (int i = 0; i < elf->e_phnum; i++, ph++) {
    load_program(
      (uint32_t)ph->p_filesz,
      (uint32_t)ph->p_memsz,
      (uint32_t)ph->p_paddr,
      (uint32_t)ph->p_offset
    );
  }
}
```

load_elf() 根据 elf 文件的 program header 调用 load_program() 将各个段加载到内存中。load_program() 中的一个小细节是：filesz 和 memsz 的差值是 elf 为 .bss 节预留的空位，根据 C 程序的约定，操作系统有义务将其填充为 0。

执行完 load_elf() 后，再查看 0x8000 附近的内存，可以看到已经被填上了内容：

```bash
(gdb) p *elf64
$1 = {e_ident = "\177ELF\002\001\001\000\000\000\000\000\000\000\000",
  e_type = 2, e_machine = 62, e_version = 1, e_entry = 1049360, e_phoff = 64,
  e_shoff = 152256, e_flags = 0, e_ehsize = 64, e_phentsize = 56, e_phnum = 2,
  e_shentsize = 64, e_shnum = 17, e_shstrndx = 16}
```

```c
if (elf32->e_machine == EM_X86_64) {
  ((void(*)())(uint32_t)elf64->e_entry)();
} else {
  ((void(*)())(uint32_t)elf32->e_entry)();
}
```

最后这两行，根据架构跳转到 elf 文件中 e_entry 所保存的入口函数地址。这里使用了 C 语言的技巧，将一个地址强制转换成一个函数指针并调用从而实现跳转。