---
layout: post
date: 2022-06-01
title: "NJU-OS Lecture 29: Xv6 File System Implementation"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Review

文件系统 = 图书馆

* 目录：图书馆操作，mkdir, rmdir, link, unlink etc.
* 文件：图书，read, write, mmap etc.
* 文件描述符 (offset)：书签，lseek etc.

<!-- more -->

FAT：

| metadata | FAT  | FAT  | data clusters |
| -------- | ---- | ---- | ------------- |

FAT 区域存放文件链表的 next 指针。FAT 的一个缺点在于一个文件的信息散落在磁盘的各个地方。

UNIX：

磁盘中的一个区域存放所有文件的 inode。inode 里包括文件的几乎所有信息 (除了所有的 data block 以一个类页表的方式存储，inode 里存放了“页表”的根)，更好地利用了数据的 locality。

## `/mkfs/mkfs.c`

mkfs 的代码写的非常琐碎。RTFSC 的优雅姿势为：读代码的执行比读代码容易。

```python
# trace.py
TRACED = 'bwrite balloc ialloc iappend rinode winode rsect wsect'.split()
IGNORE = 'ip xp buf'.split()

class trace(gdb.Breakpoint):
    def stop(self):
        f, bt = gdb.selected_frame(), []
        while f and f.is_valid():
            if (name := f.name()) in TRACED:
                lvars = [f'{sym.name}={sym.value(f)}'
                    for sym in f.block()
                    if sym.is_argument and sym.name not in IGNORE]
                bt.append(f'\033[32m{name}\033[0m({", ".join(lvars)})')
            f = f.older()
        print('    ' * (len(bt) - 1) + bt[0])
        return False # won't stop at this breakpoint

gdb.execute('set prompt off')
gdb.execute('set pagination off')
for fn in TRACED:
    trace(fn)
gdb.execute('run fs.img README user/_ls')
gdb.execute('quit')
```

上面的一段代码可以帮我们自动在 `mkfs.c` 中比较重要的 API 函数上打断点，并追踪栈帧打印函数调用链和关键参数。通过阅读 `mfks.c` 的执行过程，我们可以更快地了解 mfks 大致做了哪些事情。

## Buffer Cache

内存中的 buffer cache 是磁盘的 cache，所有的读写操作都会经过 buffer cache，buffer cache 提供了和磁盘一样的接口 bread/bwrite，这样反复的读取/反复的写入不用每次都与磁盘交互，提高了效率。

具体的代码细节见 Xv6 源码解读手册。

## Log

Xv6 的 logging layer 保证了崩溃一致性。具体的代码细节可以见 Xv6 源码解读手册。