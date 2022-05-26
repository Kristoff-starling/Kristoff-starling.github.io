---
layout: post
date: 2022-05-23	
title: "NJU-OS Lecture 27: FAT and UNIX File System"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

数据结构课的一些潜藏的假设：

* random access memory，word addressing (不同于磁盘，磁盘是以块为单位访问的)
* load/store 指令和计算指令执行的代价都是 $O(1)$。

在块设备中，即使访问一个 byte 也需要将它所在的一整个 block 拿出来，因此内存中的数据结构用在磁盘上会造成极大的性能问题。<!-- more -->

块设备提供的设备抽象：

```c
struct block blocks[NBLK]; // 磁盘
void bread(int id, struct block *buf) {
  memcpy(buf, &blocks[id], sizeof(struct block));
}
void bwrite(int id, const struct block *buf) {
  memcpy(&blocks[id], buf, sizeof(struct block));
}
```

bread() 将磁盘中的一个块拉到内存里，bwrite() 将内存中的一个块写入磁盘。

我们的文件系统在 bread()/bwrite() 上一层层抽象出来：

* 我们可以在 read/write 上实现两个 API：balloc() 和 bfree()，这样我们就把整个磁盘中所有的块管理了起来。
* 在此基础上我们可以实现一个文件数据结构，它向上呈现一个可以随机访问、修改以及变长的 byte array (C++中的 vector 容器)，向下使用 balloc()/bfree() 申请 block 并把它们串起来。
* 我们可以在文件的基础上实现目录：目录本身也是一个特殊的文件 (字节序列)，目录文件中存放了它里面有哪些文件以及它们的位置索引 (将 `vector<char>` 当作 `vector<dir_entry>` 来解读)。

## File Allocation Table (FAT)

在计算机使用软盘作为存储设备的年代，我们要在很少的存储空间上实现文件系统，因此链表是不二之选。关于链表，我们有以下两种设计：

* 数据结构课上的经典设计：让每个块有数据和 next 指针两部分：

    ```c
    struct block {
       	char data[NBLOCK];
        void *next;
    }
    ```

    这是一个很自然的想法，但有一些缺陷：比如每个 block 都有一点空间要用来存储指针，所以数据的对齐会不太好。一个更致命的缺陷是：我们在块设备上并不能 $O(1)$ 地去访问 next 指针，想要读取 next 我们要把整个 block 读出来。如果我们要执行一个 `lseek(SEEK_END)`，我们得把整个文件读一遍，这非常糟糕，而且无法克服。

* 我们花费开头的几个 block，把所有的 next 指针全部存储在这些 block 中，后面的 block 专门存储数据。这个设计就非常好地利用了数据访问的局部性：当我们在随机访问一个文件是，我们需要快速通过 next 指针跳转，这种设计将 next 指针集中存放在一起，有利于提高访问效率。

    但这个设计在数据可靠性方面有比较大的问题：一方面，所有的 next 指针都存放在开头的几个 block 内，一旦这些 block 损坏，整个文件系统就完全损坏了 (每个文件的 block 链表都完全断开)，单个 block 损坏对全局的影响过大：另一方面，由于我们经常要 lseek，所以存放 next 指针的 block 又恰恰是我们读写最频繁的区域，也就是说它们损坏的几率要远高于其他 block。不过这个问题不是完全不可解决的：我们可以备份一些 next 指针数组在磁盘的其他地方，并准备一些应急预案。

    这个磁盘开头集中存放 next 指针的 block，就叫做 file allocation table (FAT)。

若要了解 FAT 文件系统的细节，最好的方法便是 RTFM。[这里](http://jyywiki.cn/pages/OS/manuals/MSFAT-spec.pdf)有 FAT 的手册。

## ext2/Unix File System

FAT 使用最简单的链表来管理每个文件的 data block，这使得大文件的随机访问比较慢。ext2 的设计一部分解决了这个问题：ext2 中的每个文件都有一个 inode：inode 存储了每个文件的 metadata，比如文件的大小、名称等，inode 会有一个区域叫做 direct block，这样对于小文件，其 data block 可以直接放在 direct block 里，对于大文件，其 data block 可以用类似页表的方式组织起来，这样随机访问效率非常高。

所有文件的 inode 在磁盘上统一连续存储，这使得 inode 的索引非常方便，因此我们的目录文件只需要存储文件名到 inode 编号的 key-value mapping 即可。