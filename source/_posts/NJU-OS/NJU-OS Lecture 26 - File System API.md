---
layout: post
date: 2022-05-21	
title: "NJU-OS Lecture 26: File System API"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Why File System?

如果设备直接将读/写/控制的接口暴露给应用程序，那么多个应用程序在并发地使用设备时很容易发生竞争。我们考虑下面的例子：<!-- more -->

```c
// printf-race.c
#include <stdio.h>
#include "thread.h"

void use_printf(const char *s) {
  printf("%s", s);
}

void use_putchar(const char *s) {
  for (; *s; s++) {
    putchar(*s);
  }
}

void (*print)(const char *) = use_printf;

void Tworker() {
  char buf[128];
  int c = gettid() % 4 + 1;
  sprintf(buf, "\033[3%dm%d\033[0m", c, c);
  while (1) {
    print(buf);
  }
}

int main(int argc, char *argv[]) {
  if (argc > 1) {
    print = use_putchar;
  }

  setbuf(stdout, NULL);
  for (int i = 0; i < 4; i++) {
    create(Tworker);
  }
}
```

在 `printf-race.c` 中，我们创建了 4 个线程，每个线程通过 ANSI Escape Code 用某种颜色打印自己的线程号。如果我们直接运行 `./printf-race`，线程会使用 use_printf() 调用库函数 printf() 输出，这时我们能看到正确的结果。但如果我们运行时添上一个参数，比如 `./printf-race xxx`，那么线程会使用 use_putchar() 输出，这时我们就会看到大量的错误输出。产生错误的原因是：当 4 个线程并发/并行地使用 putchar() 输出时，它们的输出内容是交织在一起的，这就会导致 Escape Code 被隔断，从而产生预期之外的结果。printf() 函数使用 write 系统调用来输出给定内容，write 系统调用保证了它输出所有内容这个动作是原子的。

即使我们在设备上面封装一层有原子性的 API，对于块设备我们仍然有难以解决的问题：多个应用程序都需要从磁盘中读写数据，这就像全班人要在一张很大的纸上各自写作业，那么如何保证大家写的内容不会互相覆盖？对于系统来说，如果一个程序出了 bug 会把整个磁盘全部写上垃圾，那就没得玩了。

多个进程并发地使用内存会导致隔离被打破，我们引入了虚拟内存，给每个进程一个虚拟地址空间。在这里我们的思想是相似的：文件系统向下和物理设备 (磁盘) 打交道，向上为每个程序提供一个虚拟磁盘。虚拟磁盘就是一个可以读写的动态字节序列，我们可以直接把它想象成一个 `vector<char>`。文件系统不仅需要维护这些 vector，还要负责管理虚拟磁盘的名称，负责检索和遍历。

## Virtual Disk: Naming and Management

维护虚拟磁盘最简单的方法就是存储一堆 `名字-vector<char>` 的键值对。但这样的形式非常不利于快速检索。因此文件系统采取了树形结构。

Windows 的文件系统中，每个设备驱动器是一棵树，例如 C 盘，D 盘等。所谓的 “C 盘根目录” 就是 `C:\`，根目录下面可以构建一个错综复杂的文件树。如果插入了 U 盘等外设，系统会为它分配一个新的盘符。

> **为什么没有 `A:\` 和 `B:\`？**
>
> A 盘和 B 盘是曾经的软盘。

UNIX/Linux 的文件树和 Windows 不同。Linux 中只有一个根 `/`，更多的设备依靠 Linux 的挂载机制访问。Linux 的挂载类似于“目录树的拼接”，可以指定一个目录，将一个设备的目录树挂到该目录下面。比如当前我们有一个 `disk.img` 文件，则我们使用

```bash
sudo mount disk.img /mnt
```

之后，使用 `tree /mnt` 就可以看到磁盘镜像 `disk.img` 内的目录树结构。

Linux 的 mount 命令基于 mount 系统调用实现：

```c
int mount(const char *source, const char *target,
          const char *filesystemtype, unsigned long mountflags,
          const void *data);
```

mount 系统调用的接口更加复杂，除了要指定 source 和 target 外，还要指定文件系统的类型，挂载标志位等等。Linux 的 mount 命令使用起来容易一些，因为它可以自动监测文件系统 (busybox 的 mount 没有这个功能)。

事实上，真正的 Linux 启动的过程中也是通过挂载完成了文件系统的初始化。我们在 Linux-minimal 里面模拟了这个过程：我们用 qemu 启动的 Linux 内核只包含一个非常小的“文件系统" initramfs，这个文件系统只有非常少的一些文件，而且是存放在内存中的。我们的 Linux-minimal 做了这样一个流程：

```bash
export PATH=/bin
busybox mknod /dev/sda b 8 0
busybox mkdir -p /newroot
busybox mount -t ext2 /dev/sda /newroot
exec busybox switch_root /newroot/ /etc/init
```

其中第二行的命令使用 mknod 创建了一个设备文件 `/dev/sda`，也就是真正的磁盘。第三个命令创建了一个目录 `/newroot` (在 initramfs 中)，然后我们用 mount 命令将磁盘挂载到了 `/newroot` 上，这时候我们的 `/newroot` 其实就是真正意义上的根目录了。最后我们通过 switch_root 命令把 `/` 切换到 `/newroot` 上，并删除 initramfs 中的剩下那些多余的东西。

一个比较微妙的问题是：我们对文件的抽象是磁盘上的一个虚拟磁盘，这个虚拟磁盘是在真实物理磁盘上虚拟出来的。但如果我们做挂载，那么被挂载目录的虚拟磁盘就不是在真实物理磁盘上虚拟出来的，而是在一个虚拟磁盘 (比如 `disk.img`) 上虚拟出的虚拟磁盘，嵌套了两层。物理磁盘上的虚拟磁盘和虚拟磁盘上的虚拟磁盘显然应该有一些区别。Linux 的处理方式是创建一个 loopback (回环) 设备。回环设备的驱动会把对设备的 read/write 操作转化为对文件的 read/write 操作。我们可以通过 strace 来近距离观察 mount 的过程：

```bash
openat(AT_FDCWD, "/dev/loop-control", O_RDWR | O_CLOEXEC) = 3
ioctl(3, LOOP_CTL_GET_FREE) = 1
close(3)
```

当 Linux 想要创建一个回环设备时，它会先打开设备 `/dev/loop-control`，然后通过 ioctl 获取一个当前空闲的回环设备号。这里的返回值是 1，所以待会儿创建的回环设备就是 `/dev/loop1`。

```
openat(AT_FDCWD, "/tmp/demo/disk.img", O_RDWR | O_CLOEXEC) = 3
openat(AT_FDCWD, "/dev/loop1", O_RDWR | O_CLOEXEC) = 4
ioctl(4, LOOP_SED_FD, 3)
```

可以看到，回环设备持有文件描述符 4，Linux 通过一个 ioctl 系统调用将其指向了 `disk.img` 对应的文件描述符。这样后续对回环设备的读写操作会翻译成对 `disk.img` 文件的读写操作。	

## Directory API (System Calls)

* 创建一个目录：mkdir 系统调用，同时可以设置访问权限
* 删除一个目录：rmdir 系统调用。注意 UNIX 中没有“递归删除”的系统调用。毕竟可以交给软件做的事情就不需要交给操作系统了。`rm -rf` 会遍历目录递归删除。
* 遍历一个目录：getdents 系统调用。getdents 在 glibc 中没有封装好的同名库函数。但我们可以使用 readdir() 函数。

Linux 中的一个有趣的机制是链接。Linux 中的链接分硬链接和软链接两种。

* 如果我们对一个文件创建一个硬链接，那么这相当于创建了两个“指针” (文件名)，它们指向同一个文件实体。如果修改了一个另一个呈现的内容也会变。如果我们用 `ls -i` 查看两个文件的 inode 号，我们会发现硬链接的 inode 号和原文件是一样的。事实上，在硬链接创建后，我们无法区分出哪个是原体那个是后复制的。我们可以在 Linux 中通过 `ln a b` 创建硬链接。

* 软链接相当于一个“快捷方式”，当我们访问这个文件时，其实会跳转去访问另一个文件。软链接的使用非常灵活，它可以链接到一个目录 (硬链接只能链接文件)，可以跨文件系统，甚至可以链接到一个当前不存在的目录。我们可以在 Linux 中通过 `ln -s a b` 创建软链接。

    软链接的任意性和灵活性使得我们的文件树变成了一个文件图，它甚至可以成环：

    ```bash
    #!/bin/bash
    
    # fish-dir.sh
    # Create directories
    mkdir -p A B C D E F
    
    # Create automaton
    ln -s ../B 'A/<'
    ln -s ../C 'B/>'
    ln -s ../D 'C/<'
    ln -s ../E 'A/>'
    ln -s ../F 'E/<'
    ln -s ../D 'F/>'
    ln -s ../A 'D/_'
    ```

    我们可以在文件系统中通过软链接创建一个之前 `fish.c` 中的自动机。通过访问目录来模拟转移。

每个进程都有一个当前的工作目录，可以用 `pwd` 命令查看。如果想要修改当前工作目录，则要使用 chdir 系统调用。一个有趣的事情是：像 cat, ls 等命令在 `/bin` 下都有对应的可执行文件，但 cd 是没有的，这是因为 cd 是一个 shell 内置的命令——基本上所有的命令都是先 fork 出一个子进程再执行命令对应的可执行文件，但 cd 命令要修改的是当前进程的工作目录，显然不能用这种 fork 的方式。

> **一个进程下的多个线程是共享 working directory，还是线程之间独立？**
>
> working directory 是一个环境变量 `$PWD`，环境变量是每个进程一份，因此线程之间共享 working directory。

## File API (System Calls)

我们通过 open/pipe 等系统调用可以获得文件描述符。文件描述符可以理解为指向文件的指针，供进程访问。事实上，文件描述符里还保存了当前访问文件的偏移量，比如

```c
int fd = open("a.txt", O_RDWR | O_TRUNC);
read(fd, buf, 512);
read(fd, buf, 512)
```

第一次 read 读到的是前 512 字节，第二次 read 读到的就是后 512 字节。lseek 系统调用可以修改当前的文件偏移量。

偏移量的管理其实有很多的复杂性：比如我们知道 fork 时子进程会继承父进程的文件描述符，那么如果 fork 之后父进程向文件写入 parent，子进程向文件写入 child，我们显然不希望获得 "childt"，因此在 fork 的设计中，父子进程的文件描述符共享偏移量。此外，操作系统的每个 API 都可能和其他 API 交互，open, execve, dup 等系统嗲用中偏移量的行为各不相同，open 甚至提供了一大堆 flag 设置偏移量的行为。