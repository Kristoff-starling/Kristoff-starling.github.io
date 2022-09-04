---
title: "Lecture 28: Reliability of Persistent Storage"
linktitle: '28: 存储可靠性'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 28
---

内存在掉电后丢失数据是可以被接受的，但在持久化存储设备上我们必须保证数据的可靠性。硬盘损坏虽然是小概率事件，但只要有大量的重复 (比如数据中心)，这类事情还是经常发生的。我们希望即使发生这样的事情，系统仍然能照常运转。

## RAID

Redundant Array of Inexpensive Disks 的核心思想是：把多个 (不可靠的) 虚拟磁盘虚拟成一块非常可靠且性能极高的虚拟磁盘。RAID 的虚拟化是一种“反向”的虚拟化：我们之前接触的虚拟化，比如进程把一个 CPU 分时虚拟成多个 virtual CPU，虚拟内存把一份内存通过 MMU 虚拟成多个 virtual address space，文件把一个物理设备虚拟成多个 virtual disk……而 RAID 是 $多\to 1$ 的虚拟化。

RAID 的 Fault Model 为：任何一块磁盘都可能突然坏掉，就好像“突然消失了”。

### RAID-1

最简单的想法：准备两块磁盘 $A$ 和 $B$，它们存储了同一份虚拟磁盘的镜像，这样损坏的概率就从 $p$ 变成了 $p^2$。该做法还能提升读取的效率：比如我要从虚拟磁盘中读取 10000 个 block，那么我可以从 $A$ 中读取前 5000 块，$B$ 中读取后 5000 块，只要 CPU core 个数足够，内存带宽足够，这件事是可以做到的 (不过该做法牺牲了一些存储空间)。

### RAID-0

如果我们不考虑容错，专注于提升性能：假设两块磁盘 $A$ 和 $B$，有 $A_1,A_2$，$B_1,B_2$ 四块。虚拟磁盘有 $V_1,V_2,V_3,V_4$ 四块。如果我们的映射是 $A_1\to V_1,A_2\to V_2,B_1\to A_3,B_2\to V_4$，那么如果我们读写前两个块，就只有 $A$ 在工作，$B$ 在摸鱼；如果我们采取交错映射：$A_1\to V_1,B_1\to V_2,A_2\to V_3,B_2\to V_4$，那么我们读取任意连续的两块都能让 $A,B$ 并行地工作，性能翻倍。

### (RAID-1)-0

当我们有多块盘的时候，令 $f(i,j)$ 表示第 $i$ 块物理盘的第 $j$ 个区域对应到虚拟磁盘的哪一个区域，这里的函数 $f$ 有很大的设计空间。比如当我们有 4 块盘的时候，我么可以将之前的两种设计方法综合起来，组合成一个 (RAID-1)-0。这个写法有点像函数的复合，我们每两块盘做一个 RAID-1，这样有两组 RAID-1，再把它们用 RAID-0 连起来，这样我们就兼顾了容量、容错和性能。

这个做法有一个很有意思的点：我们可以保证任意一块物理盘出错都不会丢失信息，但我们不能保证两块盘出错时不会丢失信息——如果我们在两个 RAID-1 中各丢了一块盘，那么我们仍然能恢复信息；但如果我们丢失了第一个 RAID-1 中的所有两块盘，那么我们就真的丢失了数据。如果我们认为同时丢失了两块盘属于几乎不可能发生的小概率事件，那么这个 (RAID-1)-0 的做法似乎在容错方面就有一些“性能过剩”，有没有什么非常经济实惠的做法呢？

### RAID-4

一个块的信息可以抽象成一个 bit，于是这个问题等价于我们如何用最少的冗余信息来保证一个 bit 的错误和纠正。我们容易联想到可以用校验码来完成这件事。要保证可以纠一位错，我们只需要使用奇偶检验码即可。

假设我们现在用 $A,B,C,D$ 四块盘组合出一个虚拟磁盘，则我们的映射为 $V_1\to A_1,V_2\to B_1,V_3\to C_1$，$D_1=V_1\oplus V_2\oplus V_3$。之后 $V_4,V_5,V_6$ 依次类推。这样如果 $A,B,C$ 三块盘中的某个损坏，我们可以用 $D$ 来恢复；如果 $D$ 损坏，我们没有丢失任何实际信息，再插一块新盘算一遍异或即可。 

RAID-4 在保证坏一块盘不会丢失数据的情况下将冗余数据量做到了最小，使得我们的虚拟磁盘容量大。此外，我们将 $V_1,V_2,\cdots$ 交错映射在多个盘上，这样我们在连续读取/写入时，可以同时把三块盘的性能拉满，并行度高。

但这个设计有一个微妙的缺陷：它应对随机读写的效果较差。假设我们有多次随机的写操作，我们会发现每次写操作都要奇偶校验盘 $D$ 中的数据进行修改。此外因为 $D_i=A_i\oplus B_i\oplus C_i$，所以我们还没法直接去修改 $D_i$，假设我们现在要修改 $A_1$，那么我们必须先 `bread(A1, D1)`，然后 `D1^=A1`，将 `A1` 修改为 `A1'` 后，`D1'=D1^A1'`，最后 `bwrite(A1, D1)`。每次随机写 (不论是 $A,B,C$ 中的哪一个盘)，我们都要对 $D$ 一读一写，奇偶校验盘成为了性能瓶颈。

### RAID-5

RAID-5 的设计非常聪明：我们不用单独安排一块盘作为奇偶校验盘，我们可以把奇偶检验的信息分散到各各盘上去，每块盘上既有数据也有校验：

<img src="/img/raid5.png" alt="RAID5" style="zoom: 33%;" />

在连续读写上，$n$ 块盘可以达到 $n-1$ 块的带宽 (每 $n$ 个 block 就有一个是冗余的校验 block)，在随机读上，由于每个盘都存储了数据，因此带宽接近 100%。在随机写上，校验信息的读写仍然是瓶颈，但由于校验信息的修改被平摊到了所有盘的头上，所以基本可以保证随着盘数量的增多效率提高，有 scalability。

## Crash Consistency

另一种 Fault model：磁盘并没有故障，但操作系统可能会 crash (比如掉电)。即使是文件系统中的简单操作也可能涉及若干个 block，如果我们在一轮操作的中途掉电，那么文件系统就会进入一个 inconsistent 的状态，这样的状态可能导致严重的错误或安全问题。磁盘本身无法直接支持原子操作 (all or nothing)，甚至为了效率它不会保证 bwrite 操作是按顺序进行的 (因此 block layer 还额外提供了 bflush 操作来保证数据落盘 (和并发中的 barrier(mfence) 是类似的) )。

### File System Checking (FSCK)

FSCK 的核心原理是当磁盘上出现 inconsistent 的现象时，根据磁盘上已有的信息恢复出“最可能”的数据结构。但这套方案不是完美的：有一些无法恢复的文件可能会被放入 lost&found 中，造成一些麻烦；此外，如果 fsck 执行过程中又发生掉电，可能会产生意想不到的后果。因此 FSCK 不是一个根本的解决方法。

### Journaling

我们通常看文件系统的视角，比如目录树，是文件系统的直观表示，它是 crash unsafe 的。但如果我们从状态机视角去看文件系统，我们可以把文件系统看作一系列修改操作的序列。如果我们可以把这个 append-only 的序列存储下来，我们就可以在 crash 之后恢复出文件系统。

因此我们的想法是：在数据结构修改操作发生时，先不去做实际的修改，而是记录下一条修改日志。当日志落盘后，根据日志内容修改实际的数据结构。这样如果发生 crash，我们可以重放日志 (redo log) 来恢复文件系统。

在 bread(), bwrite() 和 bflush() 三个 API 的基础上我们可以写出一个 journal.append() 的伪代码：

```python
Journal_append(operations)
    using bread() to find the end of current journal
    bwrite(Transaction_begin)
       for operation in operation: bwrite(operation)
       bflush() # 确保所有 operation 落盘
       bwrite(Transaction_end)
       bflush() # 确保 TxEnd 标记落盘
```

再之后我们就可以将日志的内容付诸实施，实施结束后删掉日志。如果实施的过程中掉电，重启后 TxEnd 标签将成为我们确认日至是否完整的唯一标志：因为我们有 bflush() 保证同步性，因此如果 TxEnd 标志存在，那么之前所有操作的信息一定已经写入了，我们可以安全地重放日志；如果 TxEnd 标志不存在，那我们就忽略这次日志，且这时日志中的操作尚未对数据结构产生真正影响。这样就做到了 all or nothing。

> **OSLAB 中的小彩蛋**
>
> 为了提高 journaling 的效率，很多系统做出了很多的优化。比如 git 采用 metadata journaling，提升效率的同时降低了一致性，这导致有时强行关闭虚拟机会导致 git repo 处于一个不一致的状态，因此 OSLAB 的 `Makefile.lab` 中添加了一条命令：
>
> ```diff
> git:
>     @git add $(shell find . -name "*.c") $(shell find . -name "*.h") -A --ignore-errors
>     @while (test -e .git/index.lock); do sleep 0.1; done
>     @(uname -a && uptime) | git commit -F - -q --author='tracer-nju <tracer@nju.edu.cn>' --no-verify --allow-empty
> +   @sync
> ```
>
> sync 的作用是 "synchronize cached writes to persistent storage"。

