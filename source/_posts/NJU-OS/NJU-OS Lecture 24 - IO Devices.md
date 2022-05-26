---
layout: post
date: 2022-05-14	
title: "NJU-OS Lecture 24: I/O Devices"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Overview

用户直接使用的其实并不是计算设备，而是 I/O 设备。CPU 只是一个无情的指令执行器，我们希望计算机可以感知外部世界的状态，并对外实施动作。

I/O 设备就是与 CPU 交换数据的一组接口。I/O 设备会提供“几条约定好功能的线”，CPU 与设备通过握手信号可以从线上读出或写入数据。通常来说，CPU 和 I/O 设备交互的数据主要有状态 (比如 CPU 查看打印机是否空闲)，指令 (比如 CPU 让显示屏发光) 和数据 (比如 CPU 告诉打印机应该打印什么)。<!-- more -->

从抽象层的角度来说，CPU 完全不需要管 I/O 设备内部是怎么实现的，CPU 可以直接使用指令 (in/out/mmio) 和设备交换数据。但各种设备的复杂性还是使得驱动代码成为操作系统内核中非常庞大、bug非常多的一部分代码。

### Example: UART

```c
#define COM1 0x3f8

static int uart_init() {
  outb(COM1 + 2, 0);   // 控制器相关细节
  outb(COM1 + 3, 0x80);
  outb(COM1 + 0, 115200 / 9600);
  ...
}

static void uart_tx(AM_UART_TX_T *send) {
  outb(COM1, send->data);
}

static void uart_rx(AM_UART_RX_T *recv) {
  recv->data = (inb(COM1 + 5) & 0x1) ? inb(COM1) : -1;
}
```

这是 AbstractMachine 中关于串口的一部分实现。可以看到 CPU 和 IO 设备交互的基本方式是：CPU 通过 in/out 指令从设备寄存器中读取信息 (这里使用了 memory mapped io，设备寄存器是地址 COM1 开始的若干内存单元)。

### Example: Keyboard

键盘相较于串口更加复杂，现在的很多键盘是可编程的，比如我们可以软件控制 ScrollLock, CapsLock, NumLock 等灯的亮暗，可以设置按下按键时产生字符的重复速度等等。事实上 AM 采用的键盘接口只提供了两个设备寄存器，一个是数据 data (0x60)，一个是 status/command 复用的寄存器 (0x64)，各种丰富的功能都靠这两个寄存器的值的组合来实现，因此这其中有大量的约定/协议。我们的驱动程序相当于要完成这样一个 API：

```c
int keyboard_handler(int *rega, int *regb);
```

只有两个参数，但要识别出各种功能，所以代码相当难写。

### Example: Disk Controller

```c
void readsect(void *dst, int sect) {
  waitdisk();
  out_byte(0x1f2, 1);          // sector count (1)
  out_byte(0x1f3, sect);       // sector
  out_byte(0x1f4, sect >> 8);  // cylinder (low)
  out_byte(0x1f5, sect >> 16); // cylinder (high)
  out_byte(0x1f6, (sect >> 24) | 0xe0); // drive
  out_byte(0x1f7, 0x20);       // command (write)
  waitdisk();
  for (int i = 0; i < SECTSIZE / 4; i ++)
    ((uint32_t *)dst)[i] = in_long(0x1f0); // data
}
```

这段代码是 AbstractMachine 中的 readsect()。它的逻辑要比串口更加复杂一些，首先调用 waitdisk() 等待设备准备就绪，然后向设备控制寄存器写入一些数据控制磁盘向我们返回我们需要的数据 (这些内容是手册规定的)，写完 command 后我们再调用 waitdisk() 等待设备准备就绪，最终把数据给读出来。

### Example: Printer

如果要设计一个打印机对外接口，最简单的实现就是一个表示状态的寄存器 status 和一个传输打印数据流的寄存器 data。在这个简单的模型下，我们也可以做到一些很优雅的事情。

PostScript 是一种描述页面布局的比较底层的编程语言。一个小例子如下：

```postscr
%!
% http://www.cs.cmu.edu/afs/andrew/scs/cs/15-463/98/pub/www/assts/ps.html

72 72 scale	% scale coordinate system so units are inches, not points
2 2 translate	% put origin 2 inches from lower left of page

/Courier findfont .6 scalefont setfont
% current font is now Courier about .6 inches high
gsave	% save graphics state (coordinate system & stuff)
.5 setgray
60 rotate
0 0 moveto (Read the friendly manual) show
grestore	% restore previous graphics state

/Helvetica-Bold findfont .2 scalefont setfont
% current font is now slanted Helvetica about .2 inches high
0 4 moveto (Read the friendly source code) show

/Times-Italic findfont .5 scalefont setfont
% current font is now italic Times about .5 inches high
1 0 0 setrgbcolor
0 6 moveto (Search the friendly Web) show

showpage
```

它可以指定画笔的位置、颜色、角度，指定字体、大小等等。由于它是矢量绘图，所以清晰度很高。

我们完全可以以 PostScript 作为标准做一套完整的工具链：比如以 Latex 为前端，写一个编译器将 Latex 编译成 PostScript，然后我们给设备传输的字节流就是 PostScript 格式的数据。硬件设备里写一个 PostScript 的解释器将 PostScript 翻译成打印机物理硬件的动作。

当然，像打印机这种有切实物理机械部件的设备，会有很多情况需要处理，比如卡纸，缺墨等等，因此打印机的手册相当相当长。

## Bus, Interrupt Controller and DMA

### Bus

如果我们的世界里只有鼠标，键盘，显示屏几个固定的 I/O 设备，那么将它们和 CPU 连起来的最简单的方式就是在 CPU 上留几个对应的接口——每个设备插一个。但 I/O 设备会越来越多，如果要让 CPU 支持未来可能出现的更多的 I/O 设备，我们就需要一个更聪明的实现方案。

我们可以假设有这样一个特殊的 I/O 设备，它上面有若干个插槽，我们可以将各式各样的 I/O 设备插在上面。这个 I/O 设备负责管理插在它上面的其他 I/O 设备。CPU 只需要和这个 I/O 设备通信，当 CPU 需要某个设备的某个寄存器的值时，这个 I/O 设备就会到对应的插槽的对应的寄存器去找数据。这样一个提供设备的注册和地址到设备的转发的特殊 I/O 设备就是总线 (bus)。

我们可以做的更彻底一点：我们把内存 DRAM 也连到总线上面，这样 CPU 就真的只需要和总线交互了。内存单元有对应的地址，我们可以给设备寄存器也编上地址，这样 CPU 和总线之间只需要一根地址线和一根数据线 (当然，其实还需要一根用于握手的控制线)，CPU 就可以通过一个简单的地址来任意读写内存和 I/O 设备上的数据。

这里有一段示例代码：

```c
// pci-probe.c
#include <am.h>
#include <klib.h>

static inline uint32_t inl(int port) {
  uint32_t data;
  asm volatile ("inl %1, %0" : "=a"(data) : "d"((uint16_t)port));
  return data;
}

static inline void outl(int port, uint32_t data) {
  asm volatile ("outl %%eax, %%dx" : : "a"(data), "d"((uint16_t)port));
}

uint32_t pciconf_read(uint32_t bus, uint32_t slot, uint32_t func, uint32_t offset) {
   uint32_t reg = (bus  << 16) | (slot << 11)
                | (func << 8)  | (offset) | 0x80000000;
  outl(0xcf8, reg);
  return inl(0xcfc);
}

int main() {
  ioe_init();
  for (int bus = 0; bus < 256; bus ++) {
    for (int slot = 0; slot < 32; slot ++) {
      uint32_t info = pciconf_read(bus, slot, 0, 0);
      uint16_t id   = info >> 16, vendor = info & 0xffff;
      if (vendor != 0xffff) {
        printf("%02d:%02d device %x by vendor %x", bus, slot, id, vendor);
        if (id == 0x100e && vendor == 0x8086) {
          printf(" <-- This is an Intel e1000 NIC card!");
        }
        printf("\n");
      }
    }
  }
}
```

该代码扫描 PCI 总线上的每个插槽并检查上面是否有设备，此外它会额外指出以太网卡。运行该程序可以看到结果

```bash
 0: 0 device 1237 by vendor 8086
 0: 1 device 7000 by vendor 8086
 0: 2 device 1111 by vendor 1234
 0: 3 device 100e by vendor 8086 <-- This is an Intel e1000 NIC card!
```

我们还可以通过 qemu 的选项来模拟出一些新的设备：`-device AC97` 选项可以模拟出一个声卡。我们添加这个选项运行后，得到的结果确实多了一个设备：

```bash
 0: 0 device 1237 by vendor 8086
 0: 1 device 7000 by vendor 8086
 0: 2 device 1111 by vendor 1234
 0: 3 device 100e by vendor 8086 <-- This is an Intel e1000 NIC card!
 0: 4 device 2415 by vendor 8086
```

### Interrupt Controller

CPU 是无情的执行指令和响应外部中断的机器。CPU 上有一个专门的 IRQ 引脚用来接收中断信号，接收到代表中断的电信号以后，CPU 会保存一些寄存器，然后根据中断向量表对应项跳转执行中断处理程序。以前 IRQ 引脚引出的线会接到一个 Intel 8259 PIC 上 (programmable interrupt controller)，这个可编程的中断控制器可以设置各种中断屏蔽，中断优先级等等。现在有了总线之后，IRQ 连到了总线上。现在的 CPU 中有一个 APIC 模块 (Advanced PIC)，其中 local APIC 每个 CPU 一个，负责处理时钟中断，IPI 等；I/O APIC 全局一个，负责处理设备中断。

> **Inter Processor Interrupt (IPI)**
>
> 在多核处理器中，一个 CPU 核可以给另一个 CPU 核发送中断。比如在开机的时候，刚开始只有一个 CPU 核被唤醒，它执行完一些初始化操作后，就需要通过 IPI 来唤醒其他的 CPU 核开始并发执行。
>
> IPI 还有更多重要的用途。假设当前有两个 CPU 核，这两个核运行了两个线程，有独立的 cache 和 TLB，但共享内存。第一个核执行了一个 mmap 操作，将内存中的某一段做了映射。但这期间如果第二个核没有任何系统调用/中断，那么第二个核的 TLB 就没有更新！多核之间的内存可见性就消失了。因此第一个核需要发送一个 IPI 给第二个核来做一些更新，这个操作叫做 TLB shotdown。

### Direct Memory Access (DMA)

中断 I/O 相较于轮询 I/O 解决了一个问题：我们的 CPU 不用浪费时钟周期轮询等待 I/O 设备准备就绪，而是可以在设备准备好的时候给 CPU 发送一个中断信号。但我们还有一个问题没有解决：向设备传输数据这件事本身也是很慢的，比如我现在要将 1GiB 的数据写入磁盘，如果我的代码是这么写的：

```c
for (int i = 0; i < 1 GB / 4; i++)
    outl(PORT, ((uint32 *)buf)[i]);
```

那这个拷贝过程要花相当长时间。事实上，由于电气特性的限制，CPU 和总线之间存在性能 gap：总线上的设备无法做到 CPU 那么快，这无疑浪费了 CPU 的时间。

我们自然的想法是：如果我们有一个小 CPU 专门帮我们做数据拷贝就好了。它不需要通用 CPU 那么复杂，它只要能做 memcpy() 就行了，这样大 CPU 就可以被解放出来干别的事情。这就是 DMA，DMA 的本质就是一个专职 memcpy() 的小 CPU。DMA 支持从 memory 到 memory，从 memory 到 device(register)，从 device(register) 到 memory 等若干种 memcpy()，现在的 DMA 控制器直接连在总线和内存上。

## GPU and Heterogeneous Computing

DMA 相当于一个专门执行 memcpy 的小 CPU。I/O 设备和计算机之间的边界逐渐模糊。我们能不能将一些其他任务交给小 cpu 做呢？显卡就是专门绘图的 I/O 设备。

让 CPU 绘图会遇到一些性能瓶颈：以 NES 为例，如果我们要实现 60fps，就要在 10K 指令内完成一帧的绘制，而一帧的像素有 60K，这是不可能完成的任务。NES 的做法是：画面其实是用若干方块拼出来的，CPU 只需要描述将哪些方块放在什么位置，将这样一个脚本 (本质上和 PostScript) 一样发送给 NES Picture Processing Unit (PPU)，让 PPU 来完成图形显示。由于 PPU 只负责绘制图形，功能单一但算力强大，所以可以支持 60fps。

现代的 GPU 是一个通用的计算设备：它是一个完整的众核多处理器系统，有很多很多 (远多于 CPU core) 的核心数量和一个很大的 memory (显存)，GPU 从 DMA 拉数据和要执行的代码放到 memory 里，然后完成计算后再将结果推出去。

Dark Silicon Age 的到来意味着功耗成为限制 CPU 性能优化的瓶颈，CPU 的频率很难再往上提，CPU core 的数量也要受到 cache coherence 等因素的制约。因此现在很多的需求都是通过开发新的 system on chip (SoC) 来实现，比如神经网络相关的 NPU 等。