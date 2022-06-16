---
layout: post
date: 2022-02-14
title: "OSTEP Chapter 38: Redundant Arrays of Inexpensive Disks (RAIDs)"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

我们对磁盘有三个维度的需求：我们希望它读写速度快 (I/O 操作速度慢，因而成为整个系统的速度瓶颈)，我们希望它容量大，我们还希望它可靠 (如果发生磁盘损坏，仍然能恢复数据)。<!-- more -->

本章节主要介绍 Redundant Arrays of Inexpensive Disks (RAIDs) 技术，它的核心思想是用多块物理磁盘去构建一个大容量的，高速的，可靠性高的磁盘。对外来看，RAID 虚拟出了一块可读可写的磁盘；在其内部，RAID 由一个非常复杂的系统构成，包含了若干个物理磁盘和一个或多个用于控制的芯片，可以说 RAID 内部就是一个小型的计算机系统。

RAID 的优点在于：它可以通过多块磁盘并行的读写来提供很好的 performance；通过叠加磁盘的数量来获得 capacity；通过存储一部分的冗余数据来保证数据的 reliability。更重要的是，RAID 可以透明地提供这些优势，这里透明的意思是对于其他硬件/OS来说，RAID 看上去就像是一整块普通的磁盘。整个系统的其他部分不需要做任何修改就可以兼容 RAID。这一点极大地提升了 RAID 的可部署性 (deployability)。用户可以放心地将自己现有的磁盘更换成 RAID，不需要担心兼容问题。

## 38.1 Interface And RAID Internals

对于上层的文件系统来说，RAID deng |看上去就像一个磁盘。和其他单块磁盘的抽象一样，RAID 对外暴露成一个线性的 block array，每个块都可读可写。

当文件系统向 RAID 发送一个 logical I/O 请求时，RAID 内部需要搞清楚这个逻辑地址对应的块究竟在哪些盘的什么位置，并通过一次或多次 physical I/O 请求来完成这次任务。RAID 的内部结构相当复杂，通常会有一个 microcontroller 执行固件代码来控制 RAID 的行为；会有 DRAM 来作为数据块的 buffer cache；有时还会有一些非易失性的存储用来进行校验计算等。RAID 有一个完整计算机系统的大部分设施 (处理器，内存，磁盘等等)，但它是一个专门运行磁盘管理程序的专用系统。

## 38.2 Fault Model

我们在这里考虑的错误模型是一个比较简单的错误模型，称为 fail-stop。在这个模型中，每块磁盘只会处于 working 或 failed 的状态。在 working 状态，磁盘一切正常，可读可写；在 failed 状态，磁盘损坏，可以理解为该磁盘内部的数据永久丢失。

在 fail-stop 模型中，我们认为我们总是可以立刻检测到磁盘的损坏，即当某块磁盘从 working 变为 failed 时，我们可以迅速发现。因此我们不用考虑一些非常微妙的 silent failure (虽然这是实际中更可能发生的错误)。

## 38.3 How To Evaluate A RAID

我们主要从三个尺度来衡量 RAID：

* 容量 (capacity)：假设 RAID 中有 $N$ 块磁盘，每块磁盘中有 $B$ 个 block，那么理论上的最大存储量是 $N\cdot B$ 个 block，但由于我们需要存储一些必要的冗余数据来保证可靠性，实际上的容量达不到这个最大值。
* 可靠性 (reliability)：我们的 RAID 系统至多可以容忍多少个磁盘同时损坏？(容忍指损坏后可以恢复数据)
* 性能 (performance)：性能是比较难衡量的一个尺度，因为它和 workload 息息相关。

## 38.4 RAID Level 0: Striping

最简单的一种设计方法是将 block 以条带状放在各个物理磁盘上，下面展示了 4 块物理磁盘时的摆放顺序：

| Disk 0 | Disk 1 | Disk 2 | Disk 3 |
| ------ | ------ | ------ | ------ |
| 0      | 1      | 2      | 3      |
| 4      | 5      | 6      | 7      |
| 8      | 9      | 10     | 11     |
| 12     | 13     | 14     | 15     |

block 以一种 round-robin 的方式放在各个磁盘上，我们称一行中的 block 为一个 stripe。这种方式的好处是我们在读写一串连续的 block 时可以达到最佳的并行性能。上面的摆放方式是 chunk size = 1 block 的方式，即每次放放一个 block 就转到下一个 disk。我们可以调整 chunk size，比如下面展示了 chunk size = 2 block 的摆放方式

| Disk 0 | Disk 1 | Disk 2 | Disk 3 |
| ------ | ------ | ------ | ------ |
| 0      | 2      | 4      | 6      |
| 1      | 3      | 5      | 7      |
| 8      | 10     | 12     | 14     |
| 9      | 11     | 13     | 15     |

> **The RAID Mapping Problem**
>
> 在 RAID 中我们总是要处理的一个问题就是 mapping problem：给定一个逻辑块号，我们要确定它在哪一个物理磁盘上，以及在物理磁盘上的偏移量。对于 chunk size = 1 block 的 RAID 0，mapping problem 是容易的：假设逻辑块号为 $A$，则
>
> ```c
> Disk   = A % number_of_disks;
> Offset = A / numbre_of_disks;
> ```

### Chunk Size

chunk size 主要影响读写的性能。小的 chunk size 可以带来更好的并行性，但每个物理磁盘上的 chunk 的个数会比较多，定位 chunk 也是需要时间代价的。

下面的讨论中默认 chunk size = 1 block。

### Back To RAID-0 Analysis

RAID-0 的 capacity 是完美的：物理磁盘的每个 block 都被用来存储信息了；RAID-0 的 reliability 是糟糕的：任何一个磁盘的任何一个部分损坏都会导致数据丢失；RAID-0 的 performance 是比较好的，基本可以达到百分之百的带宽。

### Evaluating RAID Performance

我们衡量 RAID 的性能时通常从两种尺度出发：一种是单次请求的延迟，一种是稳定状态下的吞吐率。在衡量稳定状态下的吞吐率时，我们对两种 wordload 比较感兴趣：一种是 sequential 的，即一次性读取连续的多个 block；另一种是 random 的，即多次读取少量的 block。可以预见地，因为物理磁盘有寻道时间和旋转延迟，所以 random 的 workload 下磁盘要不停地重新定位，吞吐率会比 sequential 的情况低很多。下面的分析中，我们令 $S$ 表示 sequential 的一段长的数据的平均读取速度，$R$ 表示 random 的一块短的数据的平均读取速度。

### Back To RAID-0 Analysis, Again

从单次请求延迟的角度来看， RAID-0 的延迟基本等于一块物理磁盘的延迟，因为一个 single-block request 会被 RAID 的系统定向到某一块物理磁盘上。

从稳定状态下的吞吐率来看，RAID-0 可以达到最大的带宽，因为不论是 sequential 还是 random，当读取的 block 个数足够多时，期望情况下所有的物理磁盘都在满负荷运作。即 sequential 的吞吐率为 $N\cdot S\text{ MB/s}$，random 的吞吐率为 $N\cdot R\text{ MB/s}$。

## 38.5 RAID Level 1: Mirroring

RAID 1 的基本思想是镜像：为每个 block 保存副本，这样我们就可以忍受 disk failure。下面是一个例子：

| Disk 0 | Disk 1 | Disk 2 | Disk 3 |
| ------ | ------ | ------ | ------ |
| 0      | 0      | 1      | 1      |
| 2      | 2      | 3      | 3      |
| 4      | 4      | 5      | 5      |
| 6      | 6      | 7      | 7      |

这种方式也被称为 RAID-10 (RAID-1+0)，因为它是先复制再按照 RAID-0 的方式排开。相应地也有 RAID-01 (RAID-0+1)，先排开再做镜像。我们所说的 RAID-1 通常指 RAID-10。

RAID-1 在读取一个块时有两个选择，但 RAID-1 的写入操作必须同时修改两个副本。不过 RAID-1 的设计中一个块的两个副本位于不同的物理磁盘上，对两个副本的修改可以并行进行。

### RAID-1 Analysis

* 从 capacity 的角度，RAID-1 比较糟糕：只有一半的容量真正存储了信息，剩下的一半都是副本。

* 从 reliability 的角度，RAID-1 很可靠，因为任何一个物理磁盘损坏都不会导致数据丢失。甚至在上述例子中，如果 disk 0 和 disk 2 同时损坏，RAID-1 也不会丢失数据。RAID-1 reliability 的下限是 1 个磁盘，运气好的情况下甚至可以做到 $N/2$ 块磁盘。

* 从 performance 的角度，首先考虑 latency：对于读操作来说延迟就是一块物理磁盘的延迟；写操作则有一点不同，写操作需要更新两个副本，虽然这两个写入在不同的物理磁盘上可以并行，但两块磁盘中寻道+旋转时间较长的那块磁盘将决定写操作的延迟，因此写操作延迟略高于一块物理磁盘的期望延迟。

    接下来考虑吞吐率。首先是 sequential 的 workload。写入方面，因为每个逻辑块的写入都要同时写入两个副本，所以吞吐率为 $\frac{N\cdot S}{2}$。一个有点反直觉的结论是：读取的吞吐率也是 $\frac{N\cdot S}{2}$。虽然我们每个块有两个副本，但精心安排读取顺序并不能给我们带来效率提升。比如我们要读取 0, 1, 2, 3, 4, 5, 6, 7，如果我们只使用 Disk 0 和 Disk 2，吞吐率显然为 $\frac{N\cdot S}{2}$；我们可能会想出这样的安排方案：Disk 0 和 Disk 2 读 0 和 1，Disk 1 和 Disk 3 读 2 和 3，后面依次类推，这样是不是可以用满带宽呢？实际上不然。我们考虑一个 Disk 接受到的任务，比如 Disk 0：它要读取 0, 4, 8...... 这些 block，这些 block 在 Disk 0 上不是连续存放的，所以磁盘在旋转的时候每读取一个块，就要等待一个块，磁盘划过不需要的块的时候并没有向用户输出有效的带宽，因此它实际上只输出了一半的效率。

    在 random 的 workload 下，RAID-1 的读取是完美的：Disk 2 和 Disk 4 作为 Disk 0 和 Disk 3 的副本也可以为用户提供带宽，因此聪明的选择当前空闲的磁盘可以使吞吐率达到 $N\cdot R$。写入方面，因为要同时修改两个磁盘中的副本，所以吞吐率是 $\frac{N\cdot R}{2}$。

> **The RAID Consistent-Update Problem**
>
> RAID-1 中涉及 mirroring，在写入操作时要同时更新两个副本，因此存在所谓的 consistent-update 问题：如果我们无脑地修改第一个部分再修改第二个副本，那么加入修改完第一个副本后系统掉电了，那么重启后整个磁盘将处于一个 inconsistent 的状态——两个副本的内容不一致。因此修改两个副本的操作应当是原子的。
>
> 在文件系统中我们用 journaling 的方法来解决原子性的问题，在 RAID 中该方法同样适用。我们只要保存一个修改操作的 logging，意外掉电重启后就可以根据 logging 的内容来做恢复。值得一提的是，每次写入操作都在 disk 中做 logging 的代价太大无法忍受，因此 RAID 中一般会有一小块 non-volatile 的 RAM 用于写入 logging。

## 38.6 RAID Level 4: Saving Space With Parity

镜像所需要的额外存储空间太多。事实上，我们可以使用校验码的思路来为数据保存副本，这就是 RAID-4。RAID-4 的一个例子如下：	

| Disk 0 | Disk 1 | Disk 2 | Disk 3 | Disk 4 |
| ------ | ------ | ------ | ------ | ------ |
| 0      | 1      | 2      | 3      | P0     |
| 4      | 5      | 6      | 7      | P1     |
| 8      | 9      | 10     | 11     | P2     |
| 12     | 13     | 14     | 15     | P3     |

其中 Disk 4 上存储的都是前 4 个 disk 的校验码：$P0 = 0\oplus 1\oplus 2\oplus 3$，依此类推。这样如果有任何一个磁盘损坏，将其他 4 个磁盘的数据异或起来就可以恢复该磁盘的数据。

### RAID-4 Analysis

* 从 capacity 的角度：它比 RAID-1 好很多：$\frac{N-1}{N}$ 的空间都可以用来存储有效的信息。

* 从 reliability 的角度：RAID-4 可以容忍一个磁盘的损坏。

* 从 performance 的角度：

    * 首先考虑吞吐率：sequential 的情况是比较容易分析的：读取的时候，只有 $N-1$ 个磁盘中存储的是有效数据，所以带宽为 $(N-1)\cdot S$。写入的时候，每一个 stripe ($N-1$ 个 block) 的数据可以由 $N$ 个磁盘并行写入，因此带宽也是 $(N-1)\cdot S$。random 的读取也容易分析：只有 $N-1$ 个存储了有效数据的磁盘会工作，因此带宽为 $(N-1)\cdot R$。

        困难的是 random 写入的分析。我们在修改一个 block 的时候同时也要修改它所在的 stripe 的校验 block。我们有两种思路来计算新的校验值：一是 addictive parity，即把该 stripe 中其他的 block 都读出来，然后计算新校验值并写入，该方法的问题是其代价随着 RAID 中磁盘的上升而上升 (要做很多异或)；二是 subtractive parity，即读出旧的校验值，异或掉旧的 block，再异或上新的 block 以获得新的校验值。写成公式就是 $P_{new}=P_{old}\oplus(C_{old}\oplus C_{new})$，该方法需要对校验 block 和数据 block 各做一次读和一次写。通常来说我们选择 subtractive parity，但该方法的关键问题在于不管我们修改哪个物理磁盘上的数据，都要读写校验磁盘，因此即使数据磁盘的修改可以并行，校验磁盘仍然会将整个过程变成串行，它成为了整个系统的瓶颈。这个问题被成为 small-write problem。RAID-4 在 random 写入下的带宽是 $(R/2)$，这非常糟糕，因为它不随着 RAID 规模的增大而提高。

    * 接着分析 latency：读一个 block 的延迟和一块物理磁盘上的延迟相同；写一个 block 的延迟则复杂一些：根据之前的 subtractive parity 的方法，我们要读一次写一次数据磁盘和校验磁盘，因此延迟大约是一块物理磁盘延迟的两倍。

## 38.7 RAID Level 5: Rotating Parity

RAID-5 针对 RAID-4 的 small-write problem 做出了改进，把校验 block 分散到了各个磁盘当中：

| Disk 0 | Disk 1 | Disk 2 | Disk 3 | Disk 4 |
| ------ | ------ | ------ | ------ | ------ |
| 0      | 1      | 2      | 3      | P0     |
| 4      | 5      | 6      | P1     | 7      |
| 8      | 9      | P2     | 10     | 11     |
| 12     | P3     | 13     | 14     | 15     |
| P4     | 16     | 17     | 18     | 19     |

### RAID-5 Analysis

RAID-5 很多方面的参数和 RAID-4 是差不多的。这里主要关注 RAID-5 在 random workload 下的吞吐率：

* 对于读取操作，由于现在 $N$ 个磁盘上都存储了数据，所以带宽可以达到 $N\cdot R$。
* 对于写入操作，RAID-5 相比较于 RAID-4 有很大改善。我们可以认为在 random request 足够多的情况下，所有的物理磁盘都在满负荷运转，因此带宽可以达到 $\frac{N\cdot R}{4}$，这里要除以 4 是因为一个数据块的写入要涉及 4 次 I/O 操作。

在绝大多数场合下，RAID-5 已经完全取代了 RAID-4。除了某些特殊的场景，使用者确定不会出现大量的随机读写，这时使用 RAID-4 会使磁盘在结构上简单一些。

## 38.8 RAID Comparison: A Summary

|                  | RAID-0     | RAID-1                             | RAID-4         | RAID-5         |
| ---------------- | ---------- | ---------------------------------- | -------------- | -------------- |
| Capacity         | $N\cdot B$ | $(N\cdot B)/2$                     | $(N-1)\cdot B$ | $(N-1)\cdot B$ |
| Reliability      | $0$        | $1$ (for sure)<br>$N/2$ (if lucky) | $1$            | $1$            |
| Throughput       |            |                                    |                |                |
| Sequential Read  | $N\cdot S$ | $(N\cdot S)/2$                     | $(N-1)\cdot S$ | $(N-1)\cdot S$ |
| Sequential Write | $N\cdot S$ | $(N\cdot S)/2$                     | $(N-1)\cdot S$ | $(N-1)\cdot S$ |
| Random Read      | $N\cdot R$ | $N\cdot R$                         | $(N-1)\cdot R$ | $N\cdot R$     |
| Random Write     | $N\cdot R$ | $(N\cdot R)/2$                     | $R/2$          | $(N\cdot R)/4$ |
| Latency          |            |                                    |                |                |
| Read             | $T$        | $T$                                | $T$            | $T$            |
| Write            | $T$        | $T$                                | $2T$           | $2T$           |

如果你想要极致的性能，不在乎数据的可靠性，那么就选 RAID-0；如果你在乎 random I/O 的效率且需要可靠性，那么就选 RAID-1 (代价是容量)；如果 reliability 和 capacity 你都在乎，那么就选 RAID-5 (代价是 small-write performance)。

## 38.9 Other Interesting RAID Issues

关于 RAID 还有很多有意思的问题可以研究，比如在发生 failure 的时候系统会经历怎样的运作过程，这时候的性能会有什么变化等等。此外，我们也可以提出一些更贴合实际的 fault model，以考虑到 block corruption，latent sector error 等等。甚至有人将 RAID system 放到软件层面。

## 38.10 Summary

RAID 的核心思想是将许多块不太可靠的物理磁盘组合在一起形成一个又大又快又可靠的磁盘。RAID 的选择与使用场景的 workload 息息相关，并不是说 RAID-5 就一定比 RAID-1 来的好。因此选择合适的 RAID 模型并为其调整合适的参数 (比如 chunk size，物理磁盘个数等) 是一门艺术。