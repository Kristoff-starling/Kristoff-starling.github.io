---
layout: post
date: 2022-05-17	
title: "NJU-OS Lecture 25: Device Drivers"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

在操作系统眼里，I/O 设备 = 一组寄存器 + 一个控制协议。不同设备的控制协议千差万别，如果让软件直接和这些设备寄存器打交道，即使操作系统仔细地管理好访问权限，应用程序也很容易出错。所以我们的想法是：对不同的设备做一个抽象，使得上层应用可以通过尽可能统一的接口来访问设备。<!-- more -->

## Overview

I/O 设备最基本的需求就是 read 和 write。UNIX 世界里的设备通常分为两种：

* 字符设备 (character device)，设备与 OS 之间传送的是字节流。我们可以把这种设备想象成一个管道。终端、打印机等都是典型的字符设备。

* 块设备 (block device)，我们可以把这种设备想象成一个字符数组，这样的设备通常有 persistence 的需求，比如磁盘。

> **显卡是一种什么样的设备？**
>
> 显卡中其实既有字节流 (比如控制信号修改显卡的参数)，又有字节数组 (显卡里面有显存)。(不过显存不是按照块设备的方式来抽象的)

操作 I/O 设备，我们至少需要如下的三个 API：

* read：从设备的某个指定的位置读出数据 (如果是字符设备，则是从当前的最新位置)。
* write：向设备的某个指定的位置写入数据。
* ioctl：读取/设置设备的状态 (比如对于 GPU 来说，设置分辨率，对于打印机来说，检查纸张等)。

设备驱动程序的作用之一就是将这些统一的系统调用 API 翻译成对应到具体设备的寄存器的操作。在计算机系统的世界中这样的抽象层很多：

* Shell 负责将人类发出的命令行指令翻译成底层的系统调用；

* Driver 负责将系统调用翻译成设备相关的寄存器操作；
* ……

计算机系统世界就是这样一层一层抽象垒起来的。

> **虚拟设备**
>
> 在操作系统眼里，设备就是在执行系统调用的时候运行对应的驱动程序，操作系统不关心驱动程序究竟干了什么，因此我们可以有虚拟设备的概念：驱动程序可以真的和一个物理设备进行了交互，也可以只是模拟了一些行为。
>
> `/dev/null` 就是一个典型的虚拟设备。它像一个黑洞一样，可以吃掉所有向它输出的内容。它的设备驱动程序非常简单：对于 write，不论写入的内容是什么，它直接返回一个写入成功即可。对于 read，不论要读入什么，也都直接返回。

> 设备的复杂性
>
> 虽然设备的数据传输是其“主要”的功能，但剩下的控制功能种类繁多：比如现代的键盘可以支持呼吸灯、跑马灯、按键编程，现代打印机可以控制打印质量、卡纸、清洁、自动装订等等。这些控制功能全部依赖 ioctl 系统调用，导致其中有无比复杂的 hidden specification。	

## Linux Device Drivers

这里有一个非常简单的，用于“引爆核弹”的设备驱动：

```c
// launcher.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define MAX_DEV 2

static int dev_major = 0;
static struct class *lx_class = NULL;
static struct cdev cdev;

static ssize_t lx_read(struct file *, char __user *, size_t, loff_t *);
static ssize_t lx_write(struct file *, const char __user *, size_t, loff_t *);

static struct file_operations fops = {
  .owner = THIS_MODULE,
  .read = lx_read,
  .write = lx_write,
};

static struct nuke {
  struct cdev cdev;
} devs[MAX_DEV];

static int __init lx_init(void) {
  dev_t dev;
  int i;

  // allocate device range
  alloc_chrdev_region(&dev, 0, 1, "nuke");

  // create device major number
  dev_major = MAJOR(dev);

  // create class
  lx_class = class_create(THIS_MODULE, "nuke");

  for (i = 0; i < MAX_DEV; i++) {
    // register device
    cdev_init(&devs[i].cdev, &fops);
    cdev.owner = THIS_MODULE;
    cdev_add(&devs[i].cdev, MKDEV(dev_major, i), 1);
    device_create(lx_class, NULL, MKDEV(dev_major, i), NULL, "nuke%d", i);
  }
  return 0;    
}

static void __exit lx_exit(void) {
  device_destroy(lx_class, MKDEV(dev_major, 0));
  class_unregister(lx_class);
  class_destroy(lx_class);
  unregister_chrdev_region(MKDEV(dev_major, 0), MINORMASK);
}

static ssize_t lx_read(struct file *file, char __user *buf, size_t count, loff_t *offset) {
  if (*offset != 0) {
    return 0;
  } else {
    uint8_t *data = "This is dangerous!\n";
    size_t datalen = strlen(data);
    if (count > datalen) {
      count = datalen;
    }
    if (copy_to_user(buf, data, count)) {
      return -EFAULT;
    }
    *offset += count;
    return count;
  }
}

static ssize_t lx_write(struct file *file, const char __user *buf, size_t count, loff_t *offset) {
  char databuf[4] = "\0\0\0\0";
  if (count > 4) {
    count = 4;
  }

  copy_from_user(databuf, buf, count);
  if (strncmp(buf, "\x01\x14\x05\x14", 4) == 0) {
    const char *EXPLODE[] = {
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣀⣀⠀⠀⣀⣤⣤⣄⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⣀⣠⣤⣤⣾⣿⣿⣿⣿⣷⣾⣿⣿⣿⣿⣿⣶⣿⣿⣿⣶⣤⡀⠀⠀⠀⠀",
      "    ⠀⢠⣾⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣷⠀⠀⠀⠀",
      "    ⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣶⡀⠀",
      "    ⠀⢀⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡇⠀",
      "    ⠀⢸⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡿⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠿⠟⠁⠀",
      "    ⠀⠀⠻⢿⡿⢿⣿⣿⣿⣿⠟⠛⠛⠋⣀⣀⠙⠻⠿⠿⠋⠻⢿⣿⣿⠟⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠈⠉⣉⣠⣴⣷⣶⣿⣿⣿⣿⣶⣶⣶⣾⣶⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠉⠛⠋⠈⠛⠿⠟⠉⠻⠿⠋⠉⠛⠁⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣶⣷⡆⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⢀⣀⣠⣤⣤⣤⣤⣶⣿⣿⣷⣦⣤⣤⣤⣤⣀⣀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⢰⣿⠛⠉⠉⠁⠀⠀⠀⢸⣿⣿⣧⠀⠀⠀⠀⠉⠉⠙⢻⣷⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠙⠻⠷⠶⣶⣤⣤⣤⣿⣿⣿⣿⣦⣤⣤⣴⡶⠶⠟⠛⠁⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣴⣿⣿⣿⣿⣿⣿⣷⣄⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀",
      "    ⠀⠀⠀⠀⠀⠀⠀⠀⠀⠒⠛⠛⠛⠛⠛⠛⠛⠛⠛⠛⠓⠀⠀⠀⠀⠀⠀⠀⠀⠀",
    };
    int i;

    for (i = 0; i < sizeof(EXPLODE) / sizeof(EXPLODE[0]); i++) {
      printk("\033[01;31m%s\033[0m\n", EXPLODE[i]);
    }
  } else {
    printk("nuke: incorrect secret, cannot lanuch.\n");
  }
  return count;
}

module_init(lx_init);
module_exit(lx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("jyy");
```

该驱动在 linux 系统启动后被编译成一个 `.ko` 文件后“动态加载”，其中 lx_init() 和 lx_exit() 的内容非常琐碎 (所以驱动是 bug 重灾区！)，lx_read() 和 lx_write() 是 read()/write() 系统调用会使用的驱动函数。如果我们读 `/dev/nuke0` 这个设备，会得到 “This is dangerous!"，如果我们向这个设备写入数据，lx_write() 会检查写入的数据是否是 SECRET，如果是就会打印核弹爆炸的图案。

> **为什么 Linux 中有两种 ioctl?**
>
> 在单处理器时代 Linux 中只有 ioctl，后来多处理器时代 Linux 内核加装了 big kernel lock (BKL)，ioctl 执行时默认持有 BKL，从而驱动的执行 (本身较慢) 会使得很多线程被阻塞。后来 Linux 推出了 unlocked_ioctl，高性能的驱动可以使用该 ioctl 来避免上锁，再后来 ioctl 被删除，只剩下 unlocked_ioctl。
>
> compact_ioctl 是在 64-bit 机器的兼容模式下运行 32-bit 程序时使用的 ioctl。

## Programming for GPU

GPU 使用的是一种 SIMT (single instruction multiple thread) 的架构：相当于 GPU 里有很多的 CPU core，每个 CPU core 有自己的寄存器，core 之间共享内存 (显存)，但使用同一个 "PC指针"。因此 GPU 适合处理大量计算的简单任务，众核的优势可以使得重复计算被并行分解。

```c
// mandelbrot.cu
#include <stdio.h>
#include <stdint.h>

#define MAX_ITER 100
#define DIM 12800
static uint32_t colors[MAX_ITER + 1];
static uint32_t data[DIM * DIM];

__device__ uint32_t mandelbrot(double x, double y) {
  double zr = 0, zi = 0, zrsqr = 0, zisqr = 0;
  int i;

  for (i = 0; i < MAX_ITER; i++) {
    zi = zr * zi * 2 + y;
    zr = zrsqr - zisqr + x;
    zrsqr = zr * zr;
    zisqr = zi * zi;
    if (zrsqr + zisqr > 4.0) break;
  }
  
  return i;
}

__global__ void mandelbrot_kernel(uint32_t *data, double xmin, double ymin, double step, uint32_t *colors) {
  int pix_per_thread = DIM * DIM / (gridDim.x * blockDim.x);
  int tId = blockDim.x * blockIdx.x + threadIdx.x;
  int offset = pix_per_thread * tId;
  for (int i = offset; i < offset + pix_per_thread; i++) {
    int x = i % DIM;
    int y = i / DIM;
    double cr = xmin + x * step;
    double ci = ymin + y * step;
    data[y * DIM + x]  = colors[mandelbrot(cr, ci)];
  }
  if (gridDim.x * blockDim.x * pix_per_thread < DIM * DIM
      && tId < (DIM * DIM) - (blockDim.x * gridDim.x)) {
    int i = blockDim.x * gridDim.x * pix_per_thread + tId;
    int x = i % DIM;
    int y = i / DIM;
    double cr = xmin + x * step;
    double ci = ymin + y * step;
    data[y * DIM + x] = colors[mandelbrot(cr, ci)];
  }
}

int main() {
  float freq = 6.3 / MAX_ITER;
  for (int i = 0; i < MAX_ITER; i++) {
    char r = sin(freq * i + 3) * 127 + 128;
    char g = sin(freq * i + 5) * 127 + 128;
    char b = sin(freq * i + 1) * 127 + 128;
    colors[i] = b + 256 * g + 256 * 256 * r;
  }
  colors[MAX_ITER] = 0;

  uint32_t *dev_colors, *dev_data;
  cudaMalloc((void**)&dev_colors, sizeof(colors));
  cudaMalloc(&dev_data, sizeof(data));
  cudaMemcpy(dev_colors, colors, sizeof(colors), cudaMemcpyHostToDevice);

  double xcen = -0.5, ycen = 0, scale = 3;
  mandelbrot_kernel<<<512, 512>>>(dev_data, xcen - (scale / 2), ycen - (scale / 2), scale / DIM, dev_colors);

  cudaMemcpy(data, dev_data, sizeof(data), cudaMemcpyDeviceToHost);
  cudaFree(dev_data);
  cudaFree(dev_colors);

  FILE *fp = fopen("mandelbrot.ppm", "w");
  fprintf(fp, "P6\n%d %d 255\n", DIM, DIM);
  for (int i = 0; i < DIM * DIM; i++) {
    fputc((data[i] >> 16) & 0xff, fp);
    fputc((data[i] >>  8) & 0xff, fp);
    fputc((data[i] >>  0) & 0xff, fp);
  }
  
  return 0;
}
```

`mandelbrot.cu` 绘制了一个 $12800\times 12800$ (16亿像素) 的图片，每个像素迭代 100 次。程序中的 `__device__` 使得代码可以被编译成 GPU 可以运行的语言。main() 函数中使用了几个 cuda 的 API：cudaMalloc() 在显存中申请了一些空间，cudaMemcpy() 把计算所需的数据拷贝了过去 (当然是通过 DMA)，cudaFree() 释放显存空间。这样庞大的计算量 GPU 只需要 1.7s 左右就可以完成。

GPU 上有一套完整的工具链：比如 gcc --> nvcc，objdump --> cuobjdump，gdb --> cuda-gdb，perf --> nvprof 等等。全套工具链的实现都在驱动里，因此 NVIDIA 的驱动非常复杂。

## Abstraction of Storage Devices

存储设备属于块设备，数据块 (block) 是访问的最小单元，且不支持任意的随机访问。应用程序通常通过文件系统来读写存储设备。

Linux 的 bio (Block I/O) layer 是文件系统和磁盘设备之间的接口。通常一次磁盘访问会包括多个块的读/写，因此 bio layer 中可以有一些调度策略，比如 Linux 总是优先满足 read 操作 (因为执行 read 的程序一般总是依赖数据继续执行)，延缓 write 操作 (只要没有超过 ddl)。

简单来说，block I/O layer 的 API 是 bread()，bwrite() 和 bflush()，其中最后一个用于处理一些同步问题。我们的文件系统就是在 bio API 的基础上构建一个持久数据结构，并向上层应用提供更简单易用的接口。