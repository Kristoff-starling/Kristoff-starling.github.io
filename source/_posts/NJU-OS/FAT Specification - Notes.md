---
layout: post
date: 2022-06-14
title: "FAT Specification - Notes"
categories: ["NJU-22020230 Operating Systems", "Material"]
tags: 
mathjax: true
---

[Microsoft FAT Specification](http://jyywiki.cn/pages/OS/manuals/MSFAT-spec.pdf) 的 local copy 见此处。以下是对该手册的精华部分的翻译和解读。

## Section 1: Definitions and Notations

这里只记录比较重要的定义和记号。<!-- more -->

### 1.1 Definition

* sector：可以独立于其他单元直接获取的最小单元。
* bad(defective) sector：损坏的 sector，其中的内容无法读写。
* cluster：一个 cluster 里包含若干个连续的 sector。cluster 是 allocation 的最小单位，为文件分配空间时必须分配整数个 cluster。
* volume：一段物理上连续的块地址空间 (可以理解为“全集”)。
* partition：分区，volume 中的一部分 sector。

### 1.2 Notations

$ip(x)\triangleq [x],ceil(x)\triangleq\lceil x\rceil,rem(x/y)\triangleq x\space \text{mod}\space y$。

## Section 2: Volume Structure

FAT 文件系统的结构如下：

| BPB  | ...             | BPB  | ...  | File Allocation Table | FAT copy | Root Directory                      | Data | ...                            | Dat  |
| ---- | --------------- | ---- | ---- | --------------------- | -------- | ----------------------------------- | ---- | ------------------------------ | ---- |
|      | Reserved Region |      |      | FAT Region            |          | Root Directory Region (FAT12/FAT16) |      | File & Directory (Data) Region |      |

FAT 中的数据存储是小端的。

## Section 3: Boot Sector and BPB

BPB (BIOS Parameter Block) 位于 reserved region 的第一个 sector。这个 sector 也被称为 boot sector 或者 0 号 sector。所有的 FAT 文件系统的 boot sector 中都必须有 BPB。以下的变量中以 `BPB_` 开头的属于 BPB 的内容，以 `BS_` 开头的实际上是 boot sector 的内容，不属于 BPB。

### 3.1 BPB structure common to FAT12, FAT16, and FAT32 Implementations

这部分的 BPB 结构是 FAT12, FAT16 和 FAT32 共有的。使用 C 代码可以表示如下 (注：定义结构体时使用 `__attribute__((__packed__))` 可以不让编译器自动对齐)。

```c
struct BPB_common {
	char      BS_jmpBoot[3];   /* 0  */
	char      BS_OEMName[8];   /* 3  */
	uint16_t  BPB_BytsPerSec;  /* 11 */
	uint8_t   BPB_SecPerClus;  /* 13 */
	uint16_t  BPB_RsvdSecCnt;  /* 14 */
	uint8_t   BPB_NumFATs;     /* 16 */
	uint16_t  BPB_RootEntCnt;  /* 17 */
	uint16_t  BPB_TotSec16;    /* 19 */
	uint8_t   BPB_Media;       /* 21 */     
	uint16_t  BPB_FATSz16;     /* 22 */
	uint16_t  BPB_SecPerTrk;   /* 24 */  
	uint16_t  BPB_NumHeads;    /* 26 */ 
	uint32_t  BPB_HiddSec;     /* 28 */
	uint32_t  BPB_TotSec32;    /* 32 */ 
}__attribute__((__packed__));    
```

各个字段的解释如下：

* BS_jmpBoot：包含了一个 3 个字节的 Inter x86 格式的无条件跳转指令，用于跳转到操作系统的 bootstrap code。boot code 通常位于 boot sector 的尾部，跟在 BPB 后面。两种常见的 jump instruction 的格式为：

    ```c
    jmpBoot[0] = 0xEB; jmpBoot[1] = 0x??; jmpBoot[2] = 0x90;
    jmpBoot[0] = 0xE9; jmpBoot[1] = 0x??; jmpBoot[2] = 0x??;
    ```

* BS_OEMName：占 8 个字节，可以在 FAT 的实现中被指定为任何值。其内容通常为格式化该磁盘的系统名称 (比如如果使用 Linux 的 `mkfs.fat` 命令创建磁盘镜像，该字段就会是 "mfks.fat")。

* BPB_BytsPerSec：每个 sector 的字节数。<u>该值只能是 512, 1024, 2048, 4096 中的一个</u>。

* BPB_SecPerClus：每个 cluster 的 sector 数。<u>该值必须大于 0 且是 2 的幂次</u>。因为该字段只占一个字节，因此合法的值只有 1, 2, 4, 8, 16, 32, 64, 128。

* BPB_RsvdSecCnt：reserved region 中的 reserved sector 的个数。该值可以是任何非零整数。通常情况下，这个字段用于使数据区的开头 (即 cluster #2) 地址对齐。

* BPB_NumFATs：file allocation table 的个数。通常来说 2 是一个比较推荐的数值 (为了防止磁盘受损导致的文件系统损坏)，但 1 也是允许的。

* BPB_RootEntCnt：对于 FAT12 和 FAT16，该字段表示根目录下 32-byte 目录项的个数，<u>`BPB_RootEntCnt * 32` 理应是 `BPB_BytsPerSec` 的偶数倍。在 FAT16 中，为了达到最大的兼容性，该值应当设置为 512。对于 FAT32，该值必须是 0</u>。

* BPB_TotSec16：对于 FAT12 和 FAT16，该字段表示了文件系统中 sector 的数目。该字段只有 2 个字节，如果 sector 的数目大于等于 `0x10000`，则高位部分会存储到 BPB_TotSec32 中。<u>对于 FAT32，该字段必须是 0</u>。

* BPB_Media：合法的值有 0xF0, 0xF8, 0xF9, 0xFA, 0xFB, 0xFC, 0xFD, 0xFE 和 0xFF。<u>对于不可移动的固定磁盘，0xF8 是标准值；如果是可移动设备，该字段通常是 0xF0</u>。

* BPB_FATSz16：对于 FAT12 和 FAT16，该字段表示一个 FAT 中 16-bit sector 的个数。<u>对于 FAT32，该字段必须是 0</u>。

* BPB_SecPerTrk：该字段只在一些有着特定几何特征 (一个 volume 被划分成多个轨道，有多个读写头) 的设备上有用，表示了每个 track 上 sector 的数目。该字段是为 0x13 号中断准备的。

* BPB_NumHeads：读写头数量。该字段也是为 0x13 号中断准备的。

* BPB_HiddSec：该字段表示了这个 FAT 分区后面的隐藏 sector 的数目，也是为 0x13 号中断准备的。<u>对于没有分区的设备，该字段必须为 0</u>。此外，利用该字段来对齐数据区是不正确的。

* BPB_TotSec32：对于 FAT32，该字段表示了文件系统中 sector 的数目。对于 FAT12 和 FAT16，如果 BPB_TotSec16 足够存放对应数值，则该字段为 0，否则该字段会存储数据的高位。

### 3.2 Extended BPB structure for FAT12 and FAT16 volumes

如果是 FAT12 和 FAT16，boot sector 中还有如下的一些字段 (紧跟在 BPB_common 之后)：

```c
struct BPB_16bit {
	uint8_t   BS_DrvNum;        /* 36 */          
	uint8_t   BS_Reserved1;     /* 37 */     
	uint8_t   BS_BootSig;       /* 38 */   
	uint32_t  BS_VolID;         /* 39 */ 
	char      BS_VolLab[11];    /* 43 */      
	char      BS_FilSysType[8]; /* 54 */         
	uint8_t   __padding[448];   /* 62 */       
	uint16_t  Signature_word;   /* 510 */       
}__attribute__((__packed__));
```

各个字段的解释如下：

* BS_DrvNum：0x13 号中断的驱动号，<u>设置成 0x80 或 0x00</u>。
* BS_Reserved1：保留字段，<u>设置为 0x0</u>。
* BS_BootSig：<u>如果后面两个字段 (BS_VolID 和 BS_VolLab) 中的任何一个是非零数，将该字段设置为 0x29</u>。
* BS_VolID：该字段和 BS_VolLab 共同支持了可移动设备上的 volume tracking。这些字段使得 FAT 文件系统驱动可以识别出插入到可移动驱动上的错误磁盘。<u>BS_VolID 应当通过将当前日期和时间拼接成 32 位数值的方式生成</u>。
* BS_VolLab：Volume label。该字段的内容应当和根目录下的 11 字节 volume label 相同。FAT 文件系统应当保证如果根目录下的 volume label 被创建或者修改，该字段也会被更新。如果没有 volume label，该字段会被填充为 "NO NAME    "。
* BS_FilSysType："FAT12   "，"FAT16   " 或 "FAT     " 中的一个。注意该字段只是起到一个提供信息的作用，并不用于决定 FAT 的类型。
* Signature_word：<u>偏移量为 510 的 byte 为 0x55，偏移量为 511 的 byte 为 0xaa</u>。

### 3.3 Extended BPB structure for FAT32 volumes

如果是 FAT32，boot sector 中还会有如下的一些字段 (紧跟在 BPB_common 之后)：

```c
struct BPB_32bit {
    uint32_t  BPB_FATSz32;      /* 36 */      
    uint16_t  BPB_ExtFlags;     /* 40 */       
    uint16_t  BPB_FSVer;        /* 42 */    
    uint32_t  BPB_RootClus;     /* 44 */       
    uint16_t  BPB_FSInfo;       /* 48 */     
    uint16_t  BPB_BkBootSec;    /* 50 */        
    char      BPB_Reserved[12]; /* 52 */           
    uint8_t   BS_DrvNum;        /* 64 */    
    uint8_t   BS_Reserved1;     /* 65 */           
    uint8_t   BS_BootSig;       /* 66 */           
    uint32_t  BS_VolID;         /* 67 */           
    char      BS_VolLab[11];    /* 71 */           
    char      BS_FilSysType[8]; /* 82 */           
    uint8_t   __padding[420];   /* 90 */           
    uint16_t  Signature_word;   /* 510 */           
}__attribute__((__packed__));
```

各个字段的解释如下 (和 FAT12/FAT16 相同的部分不再解释)：

* BPB_FATSz32：一个 FAT 中包含的 32-bit sector 数目。
* BPB_ExtFlags：各个位的标志信息如下
    * 0-3 位：当前 active 的 FAT 的编号，这个编号是 0-based 的，仅在 mirroring 被关闭时有效。
    * 4-6 位：reserved。
    * 7 位：该位是 0 表示 FAT 会在运行时镜像到所有的 FAT 中；该位是 1 表示只有一个 FAT 是活跃的，这个活跃的 FAT 的编号通过 0-3 位表示。
    * 8-15 位：reserved。
* BPB_FSVer：高位的 byte 存储主版本号 (major)，低位的 byte 存储小版本号 (minor)。<u>现阶段该值必须被设置为 0x0</u>。
* BPB_RootClus：该字段表示了根目录的第一个 cluster 的编号。该值通常是 2，如果有出现 cluster 损坏的情况，则该值是 2 之后第一个可用的 cluster 的编号。
* BPB_FSInfo：该字段表示了 FAT32 的 reserved area 中 FSINFO 结构体所在的 sector 编号，通常为 1。值得注意的是，在 boot sector 的备份中也会有一个 FSINTO 结构体，但主 boot sector 和备份 boot sector 的该字段会指向同一个 FSINFO 结构体，只有那个被指向的结构体才是 up-to-date 的。
* BPB_BkBootSec：<u>该字段的值是 0 或 6</u>。如果非零，它表示了 boot record 的备份所在的 sector 号 (6 号)。
* BPB_Reserved：reserved。<u>必须被设置为 0x0</u>。

### 3.4 Initialization of a FAT volume

Section 3.5 会介绍如何决定 FAT 的类型 (12/16/32)，本 section 主要介绍在 volume 初始化时如何填写 BPB 中的字段。FAT implementation 要保证可以挂载填写了合法数值的 BPB 的设备。

软盘会被格式化为 FAT12，一个简单的、事先准备好的表格决定了软盘中 BPB 的各个字段的值。注意：FAT12 要求设备容量 <= 4MB。

对于每个 sector 大小为 512B 的设备，如果设备大小 < 512MB，该设备会被格式化为 FAT16，否则会被格式化为 FAT32。覆盖默认的 FAT 类型选择是可能的。

下面的表格来自 Microsoft Corporation FAT format utility。对于 FAT16，填写 BPB_SecPerClus 的表格如下：

```c
struct DSKSZTOSECPERCLUS {
    DWORD DiskSize;       // 以512byte sector为单位
    BYTE  SecPerClusVal;
};

DSKSZTOSECPERCLUS DskTableFAT16 [] = {
  {8400,      0},  /* disks up to 4.1 MB, the 0 value for SecPerClusVal trips an error */
  {32680,     2},  /* disks up to 16 MB, 1k cluster */
  {262144,    4},  /* disks up to 128 MB, 2k cluster */
  {524288,    8},  /* disks up to 256 MB, 4k cluster */
  {1048576,  16}, /* disks up to 512 MB, 8k cluster */
  /* The entries after this point are not used unless FAT16 is forced */
  {2097152,   32},/* disks up to 1 GB, 16k cluster */
  {4194304,   64},/* disks up to 2 GB, 32k cluster */
  {0xFFFFFFFF, 0} /*any disk greater than 2GB, 0 value for SecPerClusVal trips an error */
};
```

虽然 FAT16 一般只在 <= 512 MB 的情况下使用，但表格中仍然给出了一些大于 512MB 但强制要求使用 FAT16 的设备的填写方法。该表格的使用方法是：根据当前设备的 disk size，在表格中找出第一个小于等于 disk size 的表项，并将其第二个参数作为 BPB_SecPerClus，第一个参数会被填入 BPB_TotSec16。为了使该表格中的数值可以正确工作，BPB_RsvdSecCnt 需要被设置为 1，BPB_NumFATs 需要被设置为 2，BPB_RootEntCnt 需要被设置为 512。

对于 FAT32，填写 BPB_SecPerClus 的表格如下：

```c
DSKSZTOSECPERCLUS DskTableFAT32 [] = {
  {66600,      0}, /* disks up to 32.5 MB, the 0 value for SecPerClusVal trips an error */
  {532480,     1}, /* disks up to 260 MB, .5k cluster */
  {16777216,   8}, /* disks up to 8 GB, 4k cluster */
  {33554432,   16}, /* disks up to 16 GB, 8k cluster */
  {67108864,   32}, /* disks up to 32 GB, 16k cluster */
  {0xFFFFFFFF, 64}  /* disks greater than 32GB, 32k cluster */
};
```

为了使该表格中的数值可以正确工作，BPB_RsvdSecCnt 需要被设置为 32，BPB_NumFATs 需要被设置为 2。

下面的代码解释了如何计算 BPB_FATSz16/BPB_FATSz32 字段。该代码假设 BPB_RootEntCnt, BPB_RsvdSecCnt 和 BPB_NumFATs 已经被正确设置好。代码中的 DiskSize 是以 sector 为单位的 (即 BPB_TotSec16/BPT_TotSec32 中的值)。

```c
RootDirSectors = ((BPB_RootEntCnt * 32) + (BPB_BytsPerSec – 1)) / BPB_BytsPerSec;
TmpVal1 = DskSize - (BPB_ResvdSecCnt + RootDirSectors);
TmpVal2 = (256 * BPB_SecPerClus) + BPB_NumFATs;
if (FATType == FAT32) {
    TmpVal2 = TmpVal2 / 2;
    FATSz = (TMPVal1 + (TmpVal2 – 1)) / TmpVal2;
}
if (FATType == FAT32) {
    BPB_FATSz16 = 0;
    BPB_FATSz32 = FATSz;
} else {
    BPB_FATSz16 = LOWORD(FATSz);
    /* there is no BPB_FATSz32 in a FAT16 BPB */
}
```

注：该代码中的数学运算不是完美的，在一些情况下它算出的 FAT size 会过大，但它一定不会算小，这保证了它不会出错 (过大只是会浪费一些空间)。该代码的简洁是它的优势。

### 3.5 Determination of FAT type when mounting the volume

FAT 类型唯一决定于 volume 中 cluster 的数量 (CountofClusters)。下面的步骤描述了计算 cluster 数量的过程：

* 计算根目录占据的 sector 的数量：

    ```c
    RootDirSectors = ((BPB_RootEntCnt * 32) + (BPB_BytsPerSec - 1)) / BPB_BytsPerSec;
    ```

* 计算数据区占据的 sector 的数量 (从总 sector 数量中扣除根目录和 FAT 的 sector 数量)：

    ```c
    FATSz   = (BPP_FATSz16 != 0) ? BPB_FATSz16 : BPB_FATSz32;
    TotSec  = (BPB_TotSec16 != 0) ? BPB_TotSec16 : BPB_TotSec32;
    DataSec = TotSec - (BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) + RootDirSectors);
    ```

* 计算 CountOfCluster：

    ```c
    CountofClusters = DataSec / BPB_SecPerClus;
    ```

下面的 if 语句决定了 FAT 的类型：

```c
if (CountofClusters < 4085) {/* Volume is FAT12 */}
else if (CountofClusters < 6525) {/* Volume is FAT16 */}
else {/* Volume is FAT32 */}
```

注：一个磁盘的 cluster 数目最好不要卡在边界上 (比如 4085 个 cluster 或 65525 个 cluster)，与边界值的差最好大于 16。

最大的可使用的 cluster 号是 `CountofCluster + 1`。

### 3.6 Backup BPB Structure

sector #0 的损坏会对文件系统产生毁灭性的打击，因此为 BPB 做备份是很重要的。在 FAT32 中，sector #6 必须包含 BPB 的备份。

无论是 sector #0 中的 BPB 还是 sector #6 中的备份 BPB，BPB_BkBootSec 字段中的值都是 6。

当 sector #0 无法读取时，volume repair utility 应当从 sector #6 提取信息。

## Section 4: FAT

File Allocation Table 中的每一个 entry 都代表了一个 cluster 的状态。FAT12 中每个 FAT entry 占 12 个 bit，FAT16 中每个 FAT entry 占 16 个 bit，FAT32 中每个 FAT entry 占 32 个 bit。FAT 表中的 entry 可能比可以分配的 cluster 的数目更多，那些多出来的 FAT entry 必须被置为 0。

FAT 定义了表示文件的单向链表。整个磁盘中的第一个 cluster 的编号是 2。

> 注：一个 FAT 中的目录文件本身只是一个普通文件，它有一个特殊的属性表明它自己是目录，该文件的内容是一系列的 32 字节的目录项。

FAT entry 的内容如下表：

| FAT12         | FAT16          | FAT32               | Comments                                                     |
| ------------- | -------------- | ------------------- | ------------------------------------------------------------ |
| 0x000         | 0x0000         | 0x0000000           | cluster 当前空闲                                             |
| 0x002~MAX     | 0x0002~MAX     | 0x0000002~MAX       | cluster 在使用中，FAT entry 的内容是链表中下一个 cluster 的编号。MAX 指的是最大的合法 cluster 编号 |
| (MAX+1)~0xFF6 | (MAX+1)~0xFFF6 | (MAX+1)~0xFFFFFF6   | reserved，不可使用                                           |
| 0xFF7         | 0xFFF7         | 0xFFFFFF7           | cluster 损坏                                                 |
| 0xFF8~0xFFE   | 0xFFF8~0xFFFE  | 0xFFFFFF8~0xFFFFFFE | reserved，不可使用                                           |
| 0xFFF         | 0xFFFF         | 0xFFFFFFF           | cluster 在使用中，是一个文件的最后一个 cluster               |

注：(1) FAT32 表项中的高 4 位不使用。

(2) FAT12 中 FAT 的大小不能超过 6K 个 sector，FAT16 中 FAT 的大小不能超过 128K 个 sector，FAT32 没有限制。

### 4.1 Determination of FAT entry for a cluster

给定一个合法的 cluster 号 $N$，该 section 主要介绍如何确定哪个 FAT entry 对应了这个 cluster。

#### FAT16 and FAT32

下面的代码给出了确定某一个 cluster 的 FAT entry 所在的 sector的方法：

```c
FATSz = (BBP_FATSz16 != 0) ? BPB_FATSz16 : BPB_FATSz32;
FATOffset = (FATType == FAT16) ? (N * 2) : (N * 4);
ThisFATSecNum = BPB_RsvdSecCnt + (FATOffset / BPB_BytsPerSec);
ThisFATEntOffset = REM(FATOffset / BPB_BytsPerSec);
```

注：FATX 的 FAT 是由若干个 X bit 的 FAT entry 紧密排列而成的，因此 `FATOffset` 计算出了 $N$ 号 cluster 的 FAT entry 距离 FAT 表开头的偏移字节数，用这个字节数除以 BPB_BytsPerSec 即可得到 sector 号，取余即可得到 sector 内的偏移量。

上述代码求得的 ThisFATSecNum 是 cluster 在第一份 FAT 中的 FAT entry 所在 sector 号。如果要求其他 FAT 中的 FAT entry 的 sector 号，可以参考如下公式：

```c
SectorNumber = (FatNumber * FATSz) + ThisFatSecNum;
```

将磁盘映射到内存中之后，我们可以用如下方法读取一个 FAT entry 的内容 (下面的代码中 SecBuff 是指向对应 sector 开头的指针，WORD 是 16bit，DWORD 是 32bit)：

```c
if (FATType == FAT16)
    FAT16ClusEntryVal = *((WORD *) &SecBuff[ThisFATEntOffset]);
else
    FAT32ClusEntryVal = (*((DWORD *) &SecBuff[ThisFATEntOffset])) & 0x0FFFFFFF;
```

可以用如下方法修改一个 FAT entry：

```c
if (FATType == FAT16)
    *((WORD *) &SecBuff[ThisFATEntOffset]) = FAT16ClusEntryVal;
else {
    FAT32ClusEntryVal &= 0x0FFFFFFF;
    *((DWORD *) &SecBuff[ThisFATEntOffset]) = (0xF0000000 | FAT32ClusEntryVal);
}
```

<u>注：FAT16/FAT32 的表项不能跨 sector 存储。</u>

#### FAT12

FAT12 中的每个 FAT entry 占 12bit (1.5 个字节)。所有的计算方法和 FAT16/FAT32 类似，但由于字节数不是整数，所以涉及一些微妙的运算：

```c
if (FATType == FAT12) FATOffset = N + (N / 2); // 这里的“/”是下取整

ThisFATSecNum = BPB_RsvdSecNum + (FATOffset / BPB_BytsPerSec);
ThisFATEntOffset = REM(FATOffset / BPB_BytsPerSec);
```

在 FAT12 中，一个 FAT entry 是有可能跨 sector 存储的，下面给出了判断方法：

```c
if (ThisFATEntOffset == (BPB_BytsPerSec - 1)) {
    // 该FAT entry跨sector存储
    // 最简单的应对方法是每次读取时总是读取连续的两个sector:N和N+1(除非N是最后一个sector)
}
```

FAT12 中读取 FAT entry 内容的方法也有一些微妙：因为每个 FAT entry 占 12 个 bit，所以两个 FAT entry 正好占据 3 个字节。我们计算出的 ThisFATEntOffset 对应的是该 FAT entry 所在的第一个 byte 的地址，这意味着对于奇数 cluster 而言 FAT entry 是从 ThisFATEntOffset 的开头开始的，对于偶数 cluster 而言 FAT entry 是从 ThisFATEntOffset 所在字节的后 4 个 bit 开始的，因此代码逻辑如下 (注：`>>` 是逻辑右移)：

```c
FAT12ClusEntryVal = *((WORD *) &SecBuff[ThisFATEntOffset]); // 先读出16个bit
if (N & 0x0001)
    FAT12ClusEntryVal >>= 4;     // 16个bit中只有前12个bit是表项，低位4个bit应当清空
else
    FAT12ClusEntryVal &= 0x0FFF; // 16个bit中只有后12个bit是表项，高位4个bit应当清空
```

修改一个表项的方法为：

```c
if (N & 0x0001) {
    FAT12ClusEntryVal <<= 4;
    *((WORD *) &SecBuff[ThisFATEntOffset]) &= 0x000F;
}
else {
    FAT12ClusEntryVal &= 0x0FFF;
    *((WORD *) &SecBuff[ThisFATEntOffset]) &= 0xF000;
}
*((WORD *) &SecBuff[ThisFATEntOffset]) |= FAT12ClusEntryVal;
```

### 4.2 Reserved FAT entries

FAT 的前两个 FAT entry 是 reserved 的。

第一个 FAT entry (FAT[0]) 的内容为：低 8 位保存了 BPB_Media 的值，剩下的位置为 1。

第二个 FAT entry (FAT[1]) 的内容位：

* 对于 FAT12：该 entry 保存了 EOC mark。
* 对于 FAT16 和 FAT32：MS Windows 的设备驱动将最高的两个位作为 dirty volume 标志位，剩下的其他位置为 1。这里的高 2 位指的是 FAT16 中的 0x8000 (ClnShutBitMask) 和 0x4000 (HrdErrBitMask)，FAT32 中的 0x08000000 (ClnShutBitMask) 和 0x04000000 (HrdErrBitMask)。
    * ClnShutBitMask 位：如果这个位是 1，那么这个 volume 当前是干净的，可以被挂载并访问；如果这个位是 0，那么这个 volume 是 dirty 的，FAT 文件系统驱动无法正确地解挂载这个 volume，该 volume 的内容应当被扫描一遍以确定是否有 metadata 的损坏。
    * HrdErrBitMask 位：该位是 1 说明没有遇到任何 read/write 错误。该位是 0 说明自上一次挂载以来 FAT 文件系统驱动遇到过 read/write 错误，可能有 sector 损坏了。该 volume 的内容应当被扫描一遍以确定是否有损坏的 sector。

### 4.3 Free space determination

文件系统驱动必须扫描所有的 FAT entry 以构建一系列的文件链表和获得所有的空闲 cluster。空闲的 cluster 对应  的 FAT entry 的值为 0。空闲 cluster 在 FAT 中并没有以一个链表的形式串起来，但在 FAT32 中，BPB_FSInfo sector 可能包含了空闲 cluster 的数目。

### 4.4 Other points to note

* FAT 的实现不应当对 `(CountOfCluster + 1)` FAT entry 之后的内容有任何的假设 (这个 entry 不一定在一个 sector 的结尾处)。并且在格式化时，FAT 的最后一个 sector 中最后一个 FAT entry 之后的部分应被置为 0。

* 每个 FAT 包含的 sector 数可能比它实际需要的 sector 数多，这意味着 FAT 的最后可能会包含若干完全没有使用的 sector。驱动实现应当通过 CountOfCluster 来确定 FAT 中最后一个 valid sector 的编号。最后一个 valid sector 后面的 sector 应当全部被置为 0。

## Section 5: File System Information (FSInfo) Structure

FSInfo 结构体只在 FAT32 中有。该结构体应当在文件系统初始化时设置好并放在 sector #1 中 (即紧接着 BPB)，该结构体的备份存放在 sector #7 中。

注：FSInfo 结构体中的所有信息都是建议性的，文件系统驱动应当在初始化时填好 FSInfo 的信息，但并不一定需要在磁盘读写的过程中动态更新这个结构体 (尽管手册建议这么做)。

FSInfo 的各个字段如下：

```c
struct FSInfo {
    uint32_t FSI_LeadSig[4]; /*   0 */        
    uint8_t  FSI_rsvc1[480]; /*   4 */    
    uint32_t FSI_StrucBig;   /* 484 */      
    uint32_t FSI_Free_Count; /* 488 */        
    uint32_t FSI_Nxt_Free;   /* 492 */       
    uint8_t  FSI_rsvd2[12];  /* 496 */   
    uint32_t FSI_TrailSig;   /* 508 */      
};
```

各个字段的解释如下：

* FSI_LeadSig：值为 0x41615252，用于确认 FSInfo 格式的签名。
* FSI_Rsvd1：reserved，必须置为 0。
* FSI_StrucSig：值为 0x61417272，一个额外用于确认 FSInfo 格式的签名。
* FSI_Free_Count：该字段值为磁盘中空闲 cluster 的数量，如果不清楚则置为 0xFFFFFFFF。该字段必须在 volume 挂载时确定，手册建议该字段在 volume 解挂载时仍然保存正确的 count。
* FSI_Next_Free：保存了磁盘上第一个空闲 cluster 的编号，如果不清楚则置为 0xFFFFFFFF。该字段必须在 volume 挂载时确定，手册建议该字段在 volume 解挂载时仍然保存正确的编号。
* FSI_Rsvd2：reserved，必须置为 0。
* FSI_TrailSig：值为 0xAA550000，用于确认 FSInfo 格式的签名

## Section 6: Directory Structure

FAT 的目录是一种特殊的文件，它像是一个容器，里面装了子目录和文件的相关信息。目录文件由一系列的 32 字节的目录项 (directory entry) 构成，每个目录项描述了一个子目录或一个文件。

目录项的内容可以用如下结构体描述：

```c
struct Dirent {
    char     DIR_Name[11];     /* 0  */            
    uint8_t  DIR_Attr;         /* 11 */        
    uint8_t  DIR_NTRes;        /* 12 */         
    uint8_t  DIR_CrtTimeTenth; /* 13 */                
    uint16_t DIR_CrtTime;      /* 14 */           
    uint16_t DIR_CrtDate;      /* 16 */           
    uint16_t DIR_LstAccDate;   /* 18 */              
    uint16_t DIR_FstClusHI;    /* 20 */             
    uint16_t DIR_WrtTime;      /* 22 */           
    uint16_t DIR_WrtDate;      /* 24 */           
    uint16_t DIR_FstClusLO;    /* 26 */             
    uint32_t DIR_FileSize;     /* 28 */            
}__attribute__((__packed__));
```

各个字段的解释如下：

* DIR_Name：“短”文件名，不超过 11 个字节。

* DIR_Attr：文件的属性，合法的属性值如下所示

    | ATTR_READ_ONLY     | ATTR_HIDDEN      | ATTR_SYSTEM        | ATTR_VOLUME_ID |
    | ------------------ | ---------------- | ------------------ | -------------- |
    | 0x01               | 0x02             | 0x04               | 0x08           |
    | **ATTR_DIRECTORY** | **ATTR_ARCHIVE** | **ATTR_LONG_NAME** |                |
    | 0x10               | 0x20             | 0x0F (前四个的或)  |                |

    DIR_Attr 的高两个 bit 保留，必须被置为 0。

* DIR_NTRes：reserved，必须置为 0。

* DIR_CrtTimeTenth：创建时间 (低位)，以 1/10 秒为单位，合法范围是 `0<=DIR_CrtTimeTenth<=199`。

* DIR_CrtTime：创建时间 (高位)，颗粒度为 2s。

* DIR_CrtDate：创建日期。

* DIR_LstAccDate：上一次 access 该文件的日期。这里的 access 指的是对该文件/目录的 read/write 操作。该字段必须在文件修改时更新 (即写操作时)，填写的日期必须与 DIR_WrtDate 相同。

* DIR_FstClusHI：该文件/目录的第一个 cluster 的编号的高 16 位。该字段仅对 FAT32 有效，FAT12/FAT16 中该字段必须置为 0。

* DIR_WrtTime：上一次修改该文件的时间。在文件刚刚创建的时候，该字段的值应当与 DIR_CrtTime 一样。

* DIR_WrtDate：上一次修改该文件的日期。在文件刚刚创建的时候，该字段的值应当与 DIR_CrtDate 一样。

* DIR_FstClusLO：该文件/目录的第一个 cluster 的编号的低 16 位。

* DIR_FileSize：文件大小，以字节为单位。

### 6.1 File/Directory Name (field `DIR_Name`)

DIR_Name 共有 11 个字节，分为两个部分：8 字节的 main part 和 3 字节的扩展名。如果某一个部分字节数不足，后面会用空格补足长度。

以下是一些注意点：

* main part 和扩展名之间默认有一个 "."，这个点不会存储在 DIR_Name 中。
* `DIR_Name[0] == 0xE5` 代表这个目录项是空闲的。不过在 KANJI (日本语) 中 0xE5 这个字符是存在的，因此 KANJI 下 0xE5 会用 0x05 来代替。文件系统解析文件名时如果遇到首字节是 0x05 且字符集是 KANJI 时要将 0x05 替换成 0xE5 再返回。
* 除了 0xE5，`DIR_Name[0] == 0x00` 也代表这个目录项是空闲的。它与 0xE5 的不同在于 0x00 意味着后面跟着的所有目录项也都是空闲的。
* `DIR_Name[0]` 不能是 0x20，即文件名不能以空格开头。
* 一个目录下的所有名字都必须是唯一的。

对于文件中字符的限制如下：

* 小写字母不允许出现。
* 小于 0x20 的字符不允许出现 (除了上面提到的 KANJI 的 0x05)。
* 0x22, 0x2A, 0x2B, 0x2C, 0x2E, 0x2F, 0x3A, 0x3B, 0x3C, 0x3D, 0x3E, 0x3F, 0x5B, 0x5C, 0x5D 和 0x7C。

### 6.2 File/Directory Attributes

文件或子目录的属性值会影响文件系统驱动对该文件/子目录的操作方式。各种属性值的意义列举如下：

* ATTR_READ_ONLY (0x01)：该文件不可修改，对该文件的修改操作会以返回错误码告终。

* ATTR_HIDDEN (0x02)：除非用户显式地要求列举出隐藏文件，否则 hidden files 不应在列举目录下文件时出现 (可以理解为 `ls -a` 和 `ls` 的区别)。

* ATTR_SYSTEM (0x04)：该文件被标记为和操作系统相关的“系统文件”，除非用户显式地要求列举出系统文件，否则 system files 不应在列举目录下文件时出现。

* ATTR_VOLUME_ID (0x08)：该文件包含 volume label，这种情况下，DIR_FstClusHI 和 DIR_FstClusLO 必须都置为 0。

    只有根目录可以有一个 entry 包含该属性 (表示长文件名的 entry 不遵从该规则)。

* ATTR_DIRECTORY (0x10)：该目录项表示的是一个子目录。这种情况下，DIR_FileSize 必须为 0。

* ATTR_ARCHIVE (0x20)：当文件被创建、重命名或修改时，该属性值必须被设置，标志该文件的相关信息已经被修改。在备份时，各种工具可以利用该属性来判断一个文件是否需要备份。

### 6.3 Date/Time

DIR_CrtTime, DIR_CrtTimeTenth, DIR_CrtDate, DIR_LstAccDate 这四个字段是选填的。文件系统驱动如果不支持这些字段，则必须将其置为 0。DIR_WrtTime 和 DIR_WrtDate 这两个字段是文件系统驱动必须正确更新的。

#### Date format

DIR_CrtDate, DIR_WrtDate, DIR_LstAccDate 这三个字段需要遵从日期的格式。

* 0-4 bit：日 (1~31)
* 5-8 bit：月 (1~12)
* 9-15 bit：从 1980 年起的年份 (0~127，可以表示 1980~2107 年)

#### Time Format

DIR_CrtTime, DIR_WrtTime 这两个字段需要遵从时间格式。时间是以 2s 为颗粒度的。

* 0-4 bit：秒 (以 2s 为单位，数值范围是 0~29，可以表示 0,2,4,...,58)
* 5-10 bit：分钟 (0~59)
* 11-15 bit：小时 (0~23)

### 6.4 File/Directory Size

文件的最大大小是 0xFFFFFFFF 个字节。最大的目录大小是 $2^{21}$ 个字节。

### 6.5 Directory creation

当一个新的目录被创建时，文件系统实现必须保证以下内容：

* DIR_Attr 的 ATTR_DIRECTORY bit 要置为 1。
* DIR_FileSize 必须置为 0。
* 至少要分配一个 cluster，DIR_FstClusLO 和 DIR_FstClusHI 负责表示第一个 cluster 的编号。

(以上三条说的时该目录在其父目录的 directory entry 中的内容)

* 如果该目录只有一个 cluster，那么它对应的 FAT entry 应当被标记为 end-of-file。
* 初始分配的 cluster 的内容应当置为 0。
* 除了根目录，其他所有目录都必须在开头有如下的两个目录项：
    * `.`：该目录项表示当前目录，之前提到的 DIR_Attr, DIR_FileSize 的规则仍然需要保持，DIR_FstClusLO 和   DIR_FstClusHI 以及所有的时间、日期字段必须和当前目录的保持一致。
    * `..`：该目录项表示上一级目录，之前提到的 DIR_Attr, DIR_FileSize 的规则仍然需要保持，DIR_FstClusLO 和   DIR_FstClusHI 以及所有的时间、日期字段必须和上一级目录的保持一致 (如果上一级目录是根目录，则 DIR_FstClusLO 和 DIR_FstClusHI 必须都置为 0)。

### 6.6 Root Directory

根目录是一个特殊的 container file，在格式化时创建。

在 FAT12 和 FAT16 中，根目录必须紧跟在最后一个 FAT 后，因此根目录的第一个 sector 的编号可以按照如下方式计算：

```c
FirstRootDirSecNum = BPB_RsvdSecCnt + (BPB_NumFATs * BPB_FATSz16);
```

根目录的大小根据 BPB_RootEntCnt 字段的值计算。

在 FAT32 中，根目录是变长的，根目录的第一个 cluster 的编号存放在 BPB_RootClus 字段中。

只有根目录的 DIR_Attr 字段的值可以等于 ATTR_VOLUME_ID。

根目录没有名字 (在绝大多数操作系统中，`\` 这个名字被用作根目录)，也没有任何的时间戳 (日期/时间)，也没有 ".", ".." 这两个目录项。

### 6.7 File allocation

每个文件的目录项 (存储在包含这个文件的目录中) 都有该文件第一个 cluster 的编号。如果该文件大小为 0 则编号是 0。第一个 cluster 在 FAT 中对应的 FAT entry 要么存储了下一个 cluster 的编号，要么是一个 end-of-file 标志。

data region 的第一个 sector (cluster #2) 的编号计算方法如下：

```c
FirstDataSector = BPB_RsvdSecCnt + (BPB_NumFATs * FATSz) + RootDirSectors;
```

给定一个合法的 cluster 编号 $N$，该 cluster 的第一个 sector 的编号的计算方法如下：

```c
FirstSectorofCluster = ((N - 2) * BPB_SecPerClus) + FirstDataSector;
```

## Section 7: Long File Name Implementation (optional)

上一个 section 中提到 DIR_Name 字段的长度只有 11 个 字节，其中前 8 个字节是 main part，后三个字节是扩展名。这种目录项被称为短名目录项。但用户很多时候喜欢给自己的文件或目录起长名字，本 section 主要关注如何存储长名的目录项。

长文件名的文件的相关信息存储在一些额外的长名目录项中。注意这里的长名目录项不是独立的，而是作为短名目录项的补充。即每个文件有一个短名目录项，如果文件名过长则另有一些长名目录项专门存储名字。长名目录项必须紧靠着存放在对应的短名目录项前面 (即 长*n+短 = 一个文件/目录的描述)。

长名目录项的内容可以用如下结构体表示：

```c
struct LongDirent {
    uint8_t  LDIR_Ord;         /* 0  */       
    char     LDIR_Name1[10];   /* 1  */             
    uint8_t  LDIR_Attr;        /* 11 */        
    uint8_t  LDIR_Type;        /* 12 */        
    uint8_t  LDIR_Chksum;      /* 13 */          
    char     LDIR_Name2[12];   /* 14 */             
    uint16_t LDIR_FstClusLO;   /* 26 */             
    char     LDIR_Name3[4];    /* 28 */            
}__attribute__((__packed__));
```

各个字段的解释如下：

* LDIR_Ord：该字段表示该长名目录项是其对应的短名目录项的第几个 (一个很长的名字可能需要多个长名目录项来存储)。对于最后一个长名目录项，其 LDIR_Ord 必须或上 LAST_LONG_ENTRY (0x40)。

* LDIR_Name1：长名的第 1 ~ 5 个字符。

* LDIR_Attr：长名目录项的 LDIR_Attr 必须被设置为 ATTR_LONG_NAME：

    ```c
    ATTR_LONG_NAME = ATTR_READ_ONLY | ATTR_HIDDEN | ATTR_SYSTEM | ATTR_VOLUME_ID;
    ```

    用来判断一个目录项是否是长名目录项的 Mask 为

    ```c
    #define ATTR_LONG_NAME_MASK (ATTR_READ_ONLY | ATTR_HIDDEN | ATTR_SYSTEM | ATTR_VOLUME_ID | ATTR_DIRECTORY | ATTR_ARCHIVE);
    ```

* LDIR_Type：必须被置为 0。

* LDIR_ChkSum：针对其对应的短名目录项算出来的一个校验值。

* LDIR_Name2：长名的第 6 ~ 11 个字符。

* LDIR_FstClusLO：必须被置为 0。

* LDIR_Name3：长名的第 12 ~ 13 个字符。

### 7.1 Ordinal Number Generation

一个短名目录项对应的一系列附属的长名目录项在 LDIR_Ord 这个字段上应当满足如下要求：

* 第一个长名目录项满足 `LDIR_Ord == 1`。
* 后续的长名目录项的 LDIR_Ord 保证严格单调递增。
* 最后一个长名目录项满足 `LDIR_Ord == (N | LAST_LONG_ENTRY)`。

如果以上任何一条不满足，我们就认为该长名目录项集合被损坏了。

### 7.2 Checksum Generation

当短名目录项和长名目录项被创建的时候，我们需要创建一个 8 bit 的校验和。这个校验和根据短名目录项的 DIR_Name 字段生成：

```c
unsigned char ChkSum (unsigned char *pFcbName) {
    short FcbNameLen;
    unsigned char Sum = 0;
    
    for (FcbNameLen = 11; FcbNameLen != 0; FcbNameLen--)
        Sum = ((Sum & 1) ? 0x80 : 0) + (Sum >> 1) + *pFcbName++;
   	return Sum;
}
```

如果长名目录项中的校验和与其对应的短名目录项算出来不一致，我们就认为该长名目录项集合被损坏了。

### 7.3 Example illustrating persistence of a long name

下面的图片以 "The quick brown.fox" 这个名字为例展示了长名的存储方法：

![LongDirent](https://kristoff-starling.github.io/img/FAT_LongDirent.png)

长文件名不能超过 255 个字符 (不包括结尾的 NULL)。长文件名中的字符限制和短文件名基本一样 (见 Section 6.1)，一些额外的规则为：

* 长文件名中可以使用任意多个 `.` 字符。

* `+` `,` `;` `=` `[` `]` 这六个特殊字符不能在短文件名中出现，但长文件名中也可以使用。
* 长文件名中间允许有空格。开头和结尾的空格会被忽略。

长名目录项中使用 unicode 来存储字符，unicode 中每个字符占 16 个 bit (2 个字节)，这是与短名目录项不同的地方。另一个不同是长名目录项可以区分大小写。

### 7.4 Rules governing name creation and matching

一个目录下所有短文件名和长文件名构成的集合被称为一个 namespace。一个 namespace 下的文件名应当遵循如下规定：

* 不论是短文件名还是长文件名，在一个 namespace 中必须是全局唯一的。这里比较唯一性时忽略大小写，即如果两个名字字符一样但大小写不同也被认为是冲突的。
* 如果一个 OEM 或 unicode 字符无法被转换成系统中的合适字符，文件系统应当将其翻译成 `_`。