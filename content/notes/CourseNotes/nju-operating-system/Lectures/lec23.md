---
title: "Lecture 23: 1Bit Storage"
linktitle: '23: 1Bit的存储'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 23
---

机器指令模型只有两种当前状态：寄存器和物理内存。这些状态在物理世界中应当有真实的对应。我们的需求是：

* 可以寻址：比如可以根据编号访问某个单元格。
* 访问速度尽可能快 (甚至不惜掉电后丢失状态)。

“当前状态”的存储

* 延迟线 (delay line)：像一根存储了数据的绳子不断地转，信号会随着时间衰减，因此数据会不断通过放大器来保持不丢失。

* 磁芯内存 (Magnetic core)：类似于一个二维的小磁铁阵列，当给第 $i$ 行第 $j$ 列加电时，坐标在 $(i,j)$ 处的小磁铁就有足够大的电流可以旋转。磁芯内存是 non volatile memory - 掉电后 core 的信息是不会丢失的。但信息的改变依赖小磁铁转动这一物理动作，因此速度不够高。

    > **"Segmentation fault (core dumped)"**
    >
    > 这里的“核心已转储”的说法就是源自于磁芯内存。Linux 系统中默认的 core file size 是 0 (可以通过命令 `ulimit -a` 查看)，可以通过 `ulimit -c size` 来修改。
    >
    > 此外，在 Ubuntu 系统中，我们还需要在 root 权限下通过
    >
    > ```bash
    > echo core > /proc/sys/kernel/core_pattern
    > ```
    >
    > 修改对应文件。这样发生段错误后，系统将在程序当前目录下生成一个 core 文件。我们可以用 `segfault.c` 做一个实验。段错误后，使用 `gdb segfault core` 再次运行，就可以恢复 crash 的现场。

* SRAM/DRAM：flip-flop

## Magnetism

用小磁铁的方向来表示 0 和 1，不同方向的小磁铁会有不同方向的磁场，从而可以感应出不同方向的电流。此外，电流产生的磁场也可以用来修改小磁铁的方向。

### Magnetic Tape (1928)

磁带本质上是一个 1D 的存储设备，但我们可以通过把磁带卷起来使其变成一个近似 2D 的存储设备。因此可以在小空间内存放大量的 bit。在中间位置放一个读写头，然后通过机械转轮来定位当前读写的位置。

* 优点：便宜，容量大，存储时间相对较长
* 缺点：完全无法随机读取 (只能靠机械转轮移动)

因此磁带主要用于大量冷数据的长时间存储。

### Magnetic Drum (1932)

磁鼓一定程度解决了磁带无法随机读取的问题：将磁带一圈一圈绕在一个金属棒上，然后在棒子的侧面放很多读写头，这样读取一个 bit 的时间控制在棒子转一圈的时间之内。磁鼓的缺点是：占用的空间太大了

### Hard Disk (1956)

将磁带内卷在一个二维平面上，中间是转轴。磁盘中只有一个读写头，但读写头上有一个电动机，读写头的移动可以使其快速定位磁带的某个圈，磁带的旋转可以使圈上某个位置的信息到达读写头的位置。这样不仅速度快，而且占用体积大幅下降。

工程上对磁盘还有一些优化，比如在 z 轴上排列若干个圆盘，放置多个读写头等等。此外磁盘中的每个盘实际上不是用磁带绕出来的，而是使用了更先进的生产工艺。

* 优点：便宜，容量大，有勉强可用的随机读取能力
* 缺点：可靠性低，存在机械部件，磁头划伤盘片可能导致数据丢失。

磁盘是当今计算机系统的主力数据存储。

磁盘当中有很多事情是可以调度的，从前这个调度由操作系统负责，但现在的磁盘越做越复杂，很多内部参数已经超出了操作系统的控制，因此现代的磁盘通常会有一个片上系统 (system on chip, SoC)，这种写死的固件负责磁盘中的调度算法。

### Floppy Disk (1971)

能不能将盘片和读写头分开，从而实现数据移动？这就有了软盘。刚开始软盘真的是软的，后来的 3.5 英寸软盘为了可靠性已经是硬的了。

* 优点：价格低，数据可移动
* 缺点：由于是暴露的存储介质，所以可靠性低，且数据密度不能太大。

## Pits

在一个平整的平面上挖一些坑，这样光照在平整的地方可以很好地反射，照在坑上就不能很好地反射，从而实现 0 和 1。

### Compact Disk (CD, 1980)

在反射平面 (1) 上挖上粗糙的坑 (0)，激光扫过表面，就可以读出信息。CD 的一大问题在于只读性——挖坑容易填坑难。有一些例如 PCM (phase-change material) 的材料可以做出 rewritable disk。

* 优点：价格低，容量大，可靠性很高

    光盘的一个很有意思的优点是它很容易通过“压盘”来复制：我们可以制作一个母盘，在该挖坑的地方凸起，然后一张平整盘往上一压就是一张盘。

* 缺点：随机读取能力不高，改写困难。

## Electricity

之前的持久存储介质都有一个致命的缺陷：存在一些机械部件，机械部件可靠性不高且速度跟不上，要跟上电路的速度，我们必须用电来做存储介质。

### Solid State Drive (SSD, 1991)

Flash memory 的 floating gate 的充电放电实现了 0-1。

* 优点：大规模集成电路价格低，容量很大；flash memory 的最大优势在于读写速度，而且它有一个不讲道理的特性：容量越大速度越快 (电路级并行)；可靠性非常高：没有机械部件，所以可以随便摔。

* 缺点：几乎无。

### USB Flash Disk (1999)

迅速击败软盘，成为人手 $n$ 个的移动存储设备。

---

Flash memory 有一个关键的问题：放电会放不干净，一个 cell 经过了成千上万次读写操作后，剩余的电子就会多到好像是 1 的状态，这个 cell 成为了一个 dead cell。

该问题的解决方法类似于虚拟内存：在 OS 眼中不论是 SSD 还是 HDD 都类似于一个大数组，但事实上 SSD 里面有一个小型的计算机系统：系统会根据每个 cell 的读写次数，将所谓的 “100号cell" 映射到一个其他的 cell，就像一个 MMU 一样。这样可以保证每个 cell 使用次数差不多，也可以绕开一些 dead cell。