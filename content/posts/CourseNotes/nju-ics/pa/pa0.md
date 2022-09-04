---
title: "PA0 - 世界诞生的前夜：开发环境配置"
linktitle: 'PA0'
type: docs
draft: false
featured: false

math: true

menu:
    njuics-pa:
        parent: Contents
        weight: 1
---

## 实验进度

我完成了所有安装任务，完成了必答题，并针对思考题和遇到的其他问题进行了资料搜索和思考。

## 必答题

我按照实验要求编写了 hello.c 并编写 Makefile 文件进行编译，并且学习了 GDB 的使用。这些内容无法直接呈现，故省略。

### 关于《提问的智慧》

我们从小被要求养成提问的习惯，提问被认为是有好奇心或者善于探索的象征，但我认为背后有思考的提问才是真正有价值的。《提问的智慧》中提及了大量的提问相关的 convention 和技巧，其核心原理都是思考。

我曾在知乎上看到一句话：“道德是用来约束自己的，不是用来约束别人的。”这句话当时被用来说明你不能强求别人写给你看的题解像喂饭一样深入浅出。我觉得这句话用在提问上也一样合适。除非你花钱雇了一个人专职为你解答问题，否则别人解答你的问题是一种义务劳动。你没有权利浪费别人的时间，别人也没有义务去回答一些愚蠢的问题或者给你保姆级的教程。我总结了一些《提问的智慧》中我觉得比较精华的内容罗列如下：

* **不要问一些直接 STFW/RTFM 就能轻松获得答案的知识性问题**，他们通常具有 "xxx是什么？"“xxx怎么用” 等格式。
* 作为一个计算机界的后辈，你遇到的问题有极大的概率是已经有无数人遇到过的问题。一些经典的问题通常可以在 Stack Overflow 等知名平台上找到解答，所以**你应该确保你的问题没有现成可查的解决方案**。
* 你应该**尽可能详细地刻画你的问题**，比如写下你已经用过哪些方法但失败了，也可以贴一些截图。一方面，更详细的信息可以让别人更好地了解你的情况，提供帮助；另一方面，这可以体现你为了解决问题已经付出过你能力范围内的努力。
* **机器永远是对的**，不要说“我觉得我的gcc有问题/我觉得我的电脑坏了”之类的话（这是我在帮助上“程序设计基础”课的同学解决问题时经常听到的话）。另外，你手上的源码有很大概率经过了很多人的验证，所以**不要轻易宣布自己发现了 bug**。
* 作为一个提问者你不能显得趾高气昂，在提问中**适当使用委婉的词汇**可以使你的态度更加诚恳，但也不用过度使用以至于语句冗余。
* 对于我们这样的初学者来说问题可能还没有领域之分，但在日后如果提出一些分类较细的问题，需要注意提问的平台是否合适。

总之，想要提出一个好的问题，你需要在遇到困难时先主动自己解决，尝试能力范围内可以使用的方法，并且在你的提问中展现这个过程。这样蕴含了你的思考的问题才是有价值的。

---

以下是我的自由报告内容，我在做实验的过程中详细记录了自己遇到的所有问题和查阅的所有资料。这些笔记按照章节排列，每章通常有两个部分：

* 思考题：包含笔者针对思考题查阅的资料和我尝试给出的答案。
* 补充：这里的内容是没有在讲义中提及的问题，以及讲义中建议自学的东西。

## Installing GNU/Linux

### 补充

在安装系统时遇到问题：

> Executing grub_install failed. This is a fatal error.

重启后直接进入 GNU/grub 命令行，说明系统引导出现了问题。经过排查发现使用了与硬盘不匹配的引导方式。

硬盘分区一般有两种格式：MBR 和 GPT。前者是传统的模式，后者较新。笔者因为在一套老式电脑中安装系统所以遇到了这种格式的硬盘。

#### MBR+LegacyBIOS

MBR分区格式下最多支持四个主分区。如果想划分更多的分区，可以将其中一个主分区变为扩展分区，然后在扩展分区中建立逻辑分区。逻辑分区一般从 5 开始编号。另外 MBR 分区表对硬盘的容量大小有限制。

MBR （Master Boot Record，主引导记录）存放在第一个扇区（一个扇区512字节）中，这个扇区不属于任何一个分区。这个扇区中包含如下内容：

* Bootloader：446字节，内含引导程序，主要负责识别活动分区并引导系统
* DPT (Partition Table，分区表)：64字节，存储了分区信息
* Magic Number: 2字节，固定为0x55AA，结束标志字。

分区表的64个字节分成4组每组16个字节，分别记录了4个主分区的信息。

| Partition Flag  (00)            | Start CHS (01~03) | Partition Bytes (04)                     | End CHS (05~07)   | Start LBA (08~11) | Size (12~15)         |
| ------------------------------- | ----------------- | ---------------------------------------- | ----------------- | ----------------- | -------------------- |
| 00: 不可引导   <br>80: 可以引导 | 分区开始的 CHS 值 | 分区类型<br>83/linux 82/swap 8e/LVM etc. | 分区结束的 CHS 值 | 相对的起始扇区号  | 该分区拥有的扇区数量 |

CHS 和 LBA 是两种不同的硬盘寻址方式。

* CHS寻址模式将硬盘划分为磁头 (Heads)，柱面 (Cylinder) 和扇区 (Sector) 。

    * 磁头：每张磁盘的正反面各有一个磁头，一个磁头对应了一个磁面。
    * 柱面：所有磁片中半径相同的同心圆磁道构成柱面。
    * 扇区：将圆环形磁道划分成若干个区段，就是扇区。每个扇区的容量为 512 字节。

    CHS 部分共使用了 3 个字节 24 个二进制位，因此寻址范围是 $2^{24} bit$ ，约为 8G。这种方式较为古老现在已经几乎不用。

* LBA (Logic Block Addressing) 模式中地址不再表示实际的物理地址，而是将 CHS 地址转换成一种唯一的线性地址。由于 LBA 部分共有 32 个二进制位，因此寻址范围扩大到约 2.2TB。这就是前文所说的 MBR 硬盘分区的容量限制，对于更大的硬盘，LBA寻址方式并不能表示其地址。

MBR分区格式的硬盘使用 Legacy BIOS 来引导操作系统。安装系统时应当在 BIOS 中调整引导方式为 `Legacy only` 或将 Legacy 模式的优先级放在 UEFI之前。

以 Windows 系统为例，传统 Legacy 引导操作系统时，会通过活动分区（MBR只允许有一个活动分区）下的 bootmgr（启动管理器）文件导入根目录下 boot 文件夹内的 BCD 文件，然后 BCD 文件根据自身的配置内容加载系统启动文件 `根目录\Windows\system32\winload.exe`来启动系统。

Linux 使用的引导程序是 GRUB。

#### GPT+UEFI

GPT分区格式没有4个主分区的限制，且没有主分区、扩展分区、逻辑分区这些概念的区分，对于硬盘的容量也（几乎）没有限制。与GPT对应的UEFI启动方式相较于LegacyBIOS，省去了加电自检的环节，可以更快地引导系统启动，且界面支持鼠标甚至蓝牙等。这都是GPT+UEFI相较于传统的方式优秀的地方。

GPT格式的硬盘有如下部分：

* PMBR (Protective MBR)：占据第一个扇区，其目的是防止只能识别MBR格式的工具误识别并写入硬盘。
* Primary GPT Header：占据第二个扇区
* Partition Entries：占据第三、第四个扇区
* Partitions
* Backup Partition Entries/Primary GPT Header：对之前的信息进行备份。如果表头遭到了误删等操作可以利用备份进行恢复，增加了安全性。

UEFI引导需要硬盘中有一个 EFI 分区。EFI 分区中存放了引导程序，如 Windows 的 bootmgfw.efi，Linux 的 grubx64.efi 等。

#### 装机注意事项

为了同时兼容 MBR 和 GPT 型的硬盘，现在的 BIOS 同时提供 Legacy 和 UEFI 两种启动方式，因此需要根据硬盘将 BIOS/UEFI 调整到正确的模式。对于 MBR 型的硬盘，不需要创建 EFI 分区（安装 Linux 的时候会警告你没有创建 EFI，但没关系），最后的启动分区默认即可。对于 GPT 型的硬盘，可以为新系统创建单独的 EFI 分区，也可以与老系统共用 EFI 分区，但得保证有 EFI 分区。如果启动分区选择了 Linux 的 EFI 分区，将由 Linux 引导 Windows，在 GNU/GRUB 界面上可以选择 Ubuntu 或者 Windows Boot Manager。

## First Exploration with GNU/Linux

### 思考题

>**Why executing the `poweroff` command requires superuser privilege?**
>
>Can you provide a scene where bad things will happen if the `poweroff` command does not require superuser privilege?

例如我有一台服务器，我作为服务器的管理者可以用root的权限对网站进行修改操作。网站的使用者注册账号之后可以用普通账户来访问允许范围内的资源。如果普通账户拥有过大的权限以至于可以修改服务器的系统文件甚至直接关机，那么网站将直接崩溃。

## Installing Tools

### 补充

在安装软件包的过程中遇到依赖问题无法安装，这是因为当前需要安装的软件包依赖其他软件包，而其他软件包在当前的源上找不到。我们可以添加更多的 Ubuntu 源，从而提供其他软件包的资源。

例如使用清华大学的源，在 `/etc/apt/sources.list` 中添加如下语句：

```
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ hirsute main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ hirsute-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ hirsute-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ hirsute-security main restricted universe multiverse
```

执行 `sudo apt-get update` 之后再下载软件包，即可正常进行。

## Configuring Vim

### Vim 的 Recording 功能

Recording 功能可以帮助我们完成重复工作。以讲义中的示例为例，欲输出

```
1
2
...
100
```

只需依次按下 `i1<ESC>q1yyp<C-a>q98@1` 即可。

按 `q` 再按下任意一个字符之后进入 Recording 模式。这里的字符相当于一个缓冲器名，表示接下来录制的这段动作将保存在这个缓冲器中，我们可以在不同的缓冲器中录制不同的动作。在示例中，动作被存入了缓冲器 `1` 中。

接下来 `yy` 命令是复制光标所在行整行，`p` 命令是将剪贴板内容粘贴到当前光标下一行并且将光标移动到下一行。`<C-a>/<C-x>` 操作可以进行简单的算数运算，对整数进行+1/-1。

再按一次 `q` 结束当前录制。想要使用之前的动作序列时，先直接按下需要执行的次数，然后按下 `@` ，再按下指定的缓冲器名即可。本例中我们直接输入了1，在录制宏的过程中完成了2，因此只需要再执行98次。

### Vim 的 Visual Block 模式

Visual Block 模式可以帮助我们对文本进行批量操作。以讲义中的示例为例，欲将下面片段每行左右对调

```bash
aaaaaaaaaaaaaaaaaaaaaaaaabbbbbbbbbbbbbbbbbbbbbbbbb
cccccccccccccccccccccccccddddddddddddddddddddddddd
eeeeeeeeeeeeeeeeeeeeeeeeefffffffffffffffffffffffff
ggggggggggggggggggggggggghhhhhhhhhhhhhhhhhhhhhhhhh
iiiiiiiiiiiiiiiiiiiiiiiiijjjjjjjjjjjjjjjjjjjjjjjjj
```

只需依次输入 `gg<C-v>24l4jd$p` 即可。

`gg` 命令可以将光标移动到文本开头。按下 `<C-v>` 进入 Visual Block 模式。 `24l4j` 将光标向右移动24格之后再向下移动4格，从而选中了左边一整块所有字符。`d` 操作删除选中的部分，并将删除的内容放入剪贴板。`$` 命令将光标移动到行末（不改变当前的 normal 模式）,`p` 粘贴，从而完成任务。

### GNU Diff Format

可以参照 [这个文档](https://www.gnu.org/software/diffutils/manual/html_node/Unified-Format.html) 来详细了解 GNU diff format。

**注意：这种格式只是一种严谨规范地向他人描述修改内容的方法，并不是要将这些内容直接写入要修改的文件！**

### 补充

若使用讲义中的方法直接将 `/etc/vim/vimrc` 复制到 `~/.vimrc`，我们在编辑保存的时候会遇到一些权限问题。

使用 `ls -l -a | grep vimrc` 命令查看 `~/.vimrc` 的详细信息，可以看到其权限信息为 `-rw-r--r--`，这说明除了创建者 `root` 之外别的用户对该文件没有写入的权限。我们可以用 `sudo chmod 664 ~/.vimrc` 命令将其权限修改为 `-rw-rw-r--` 来解决问题。

## More Exploration

### 思考题

> **消失的cd**
>
> 为什么上述基本命令除了 `cd` 之外都能找到它们的 manpage?

`cd` 并不是一条指令，如果在终端中输入 `type cd` 我们会得到 “cd is a shell builtin”。

> **Things Behind Scrolling**
>
> Why the original terminal cannot be scrolled? How does `tmux` make the terminals scrollable? How to implement a scroll bar?

留坑。盲猜最基本的滚轮功能是滚一格向上/下移动 $x$ 行，欲实现滚轮需要将终端内的历史命令和输出保存下来。

### 补充

#### 正则表达式的最基本语法

`[]` 语法

| 正则表达式 |                         含义及其说明                         |
| :--------: | :----------------------------------------------------------: |
|    `oo`    |                              oo                              |
| `[abcd]oo` |           aoo/boo/coo/doo<br>`[]` 表示匹配一个字符           |
| `[^ef]oo`  | xoo，x不为e和f<br>`^` 是取反符号，且取反中括号内所有列出的字符。 |

行首行末匹配 `^` `$`
| 正则表达式 |                         含义及其说明                         |
| :--------: | :----------------------------------------------------------: |
| `^[^A-Z]`  | 首字符不是大写字母的行<br>注意 `^` 在中括号中表示取反，不在中括号中表示行首 |
|   `\.$`    |   以 `.` 结尾的行<br>注意 `.` 有特殊含义，要用 `\` 转义。    |
|    `^$`    |                            空白行                            |

任意字符 `.`，重复字符 `*`

|  正则表达式   |                        含义及其说明                         |
| :-----------: | :---------------------------------------------------------: |
|    `g..d`     |   长度为4的以g开头d结尾的串<br>`.` 表示“一定要有一个字符”   |
|    `ooo*`     | 两个以及更多的o<br>`*` 表示将之前的一个字符重复零次或若干次 |
|    `go.*g`    |   以go开头g结尾的单词，中间可以有任意的字符（也可以为空）   |
| `[0-9][0-9]*` |                          任意数字                           |

限定次数 `{}`

| 正则表达式  |                  含义及其说明                   |
| :---------: | :---------------------------------------------: |
|  `o\{2\}`   | o连续出现两次，即oo<br>注意使用 `{}` 一定要转义 |
| `o\{2,5\}`  |    o出现两次到五次之间，即oo/ooo/oooo/ooooo     |
| `go\{2,\}d` |     开头是g结尾是d,中间出现了两次及以上的o      |

扩展正则表达式语法

| 正则表达式  |                         含义及其说明                         |
| :---------: | :----------------------------------------------------------: |
|   `go+d`    | 开头是g结尾是d，中间出现了一次及以上的o<br>`+` 表示前一个字符出现了若干次（与 `*` 的区别是非零） |
|   `go?d`    |         gd或god<br>`?` 表示前一个字符重复零次或一次          |
|  `gd|god`   |                    gd或god<br>`|` 表示或                     |
| `g(la|oo)d` |     glad/good<br>`()` 里可以罗列若干个串并用 `|` 分隔开      |
| `A(xyz)+B`  | 开头是A结尾是B中间有一个或多个xyz<br>`()+` 表示括号中的群组重复一次或多次 |

#### Standard Streams

> In Computer Programming, **standard streams** are interconnected input and ouput communication channel between a computer program and its environment when it begins execution. (From Wikipedia)

标准流包括三个：标准输入 (stdin, 0)，标准输出 (stdout, 1) 和标准错误 (stderr, 2)。默认情况下，标准输入是通过键盘输入，标准输出和标准错误都是直接打印在终端上通过屏幕显示。可以通过重定向 (redirection) 或管道 （pipeline）来修改标准流。此外，子进程继承父进程的标准流。

#### 关于 `find` 和 `xargs` 命令

在 Linux 入门教程中提到使用 `find . | grep '\.c$|\.h$' | xargs wc -l` 来统计代码的行数。笔者在本地直接使用该命令时报错：`xargs: unmatched single quote`。查阅资料得知这是因为文件名中有奇数个单引号。与此同时，文件名中的空格、斜杠等特殊字符也会导致识别错误。

配合使用 `find` 命令的 `-print0` 参数和 `xargs` 命令的 `-0` 参数可以解决这个问题。它们的作用是用一个 null character 来代替空格分隔文件名，同时可以解决文件名中的空格，斜杠等问题。但我们发现 `find . -print0` 的结果送给 `grep` 无法正常地完成筛选。

我们发现 `find` 命令自带文件名筛选功能。我们可以使用 `-name '*.cpp'` 这个选项来筛选出所有 .cpp 文件，这样使用命令

```bash
find . -name '*.cpp' -print0 | xargs -0 wc -l 
```

可以完成所有 .cpp 文件的代码行数统计，要注意 `-name` 选项后的文件名并不支持完整的正则表达式语法，根据手册，它只支持 `*` `?` 和 `[]` 语法。

如果执意想使用 `grep` 命令的话，我们可以换一个思路来解决这个问题：给文件名加上双引号来解决特殊字符。

```bash
find . -printf '"%p"\n' | grep '\.cpp"$' | xargs wc -l
```

`-printf` 参数后跟 `%p` 可以输出文件名，我们将所有的文件名用双引号括起来，加一个换行符，就可以让 `find` 在标准输出中将文件名带引号地分行输出，注意此时 `grep` 指令需要匹配行尾的 `"` 。

#### `/dev/null` , `/dev/zero` 和 `/dev/random`

`/dev/null` 是一个只写文件。如果使用 `cat /dev/null` 查看文件内容会得到空。通常可以通过将标准输出重定向到 `/dev/null` 的方法来吃掉你不想要的输出内容。虽然无法写入，但在写入时它会返回给你写入成功。

`/dev/zero` 在读取时会提供无限的空字符。可以用来覆盖文件内容或产生特定大小的空文件。

`dev/random` 在读取时会产生永不为空的随机字节数据流。

对于这些特殊文件实现方式的一个简单解释是，例如 `/dev/null` ,当内核接收到对这个文件的请求时，不需要进行任何真实的硬件操作，直接按照API返回写入成功即可。

#### Makefile

Makefile最简单的一点语法：

```
要生成的文件名:依赖文件列表
	用于生成目标文件的命令序列
```

如果连续多次执行 `make` ，会得到信息 'xxx is up to date'。这是 `make` 程序智能管理的功能，如果目标文件已经存在且修改时间后于其依赖的所有文件，`make` 会认为这个文件已经编译过了，不需要重新编译。

（即使这个文件并不是按照命令要求生成的，如在当前目录下有文件 `hello.c` 和 `Makefile`。`Makefile` 的内容如下：

```makefile
hello:hello.c
	gcc hello.c -o hello
```

此时用 `touch hello` 指令创建一个修改时间最新的 `hello` 空文件再执行 `make` ，仍然显示 ‘hello is up to date’。）

使用语句 `:PHONY xxx` 可以将 `xxx` 变成一个伪目标，这样以后在执行 `make xxx` 时不会再检查文件 `xxx` 的新旧，而是直接执行 `xxx` 规则下的命令。

## Getting Source Code for PAs

### 思考题

> ##### What happened?
>
> You should know how a program is generated in the 程序设计基础 course. But do you have any idea about what happened when a bunch of information is output to the screen during `make` is executed?

留坑。

### 补充

#### Git使用 - 关于 SSH Key 的设置

如果想要向远程仓库提交修改，必须将本机的 SSH Key 添加到受信任的 SSH Key 列表中。

如果本地没有 SSH Key，可以使用命令 `ssh-keygen -t rsa -C "email"` 命令来生成 SSH Key，其中 email 是在 `git config --global user.email` 中填写的邮箱。默认生成位置在 `~/.ssh` 中。生成后将公钥 `~/.ssh/id_rsa.pub` 中的内容填写到受信任 SSH Key 列表中即可。

#### 关于 `make menuconfig` 的报错

在 `ics2021/nemu` 下使用命令：`make menuconfig`，报错 `bison: No such file or directory` 。

报错是因为缺少 bison 工具，安装即可。对于之后的 flex 工具也是亦然。

**注：当你缺少某项工具时，不一定会返回 `command not found`，也可能会返回 `No such file or directory`!**

#### 关于软件安装的报错

在使用 `apt-get install` 安装软件时若出现形如以下的错误：

```
E: Failed to fetch http://mirrors.tuna.tsinghua.edu.cn/ubuntu/pool/main/f/flex/flex_2.6.4-8_amd64.deb  Connection timed out [IP: 2402:f000:1:400::2 80]
E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?
```

虽然可能是源中没有对应的 package，但 `Connection timed out` 也可能是因为网速不行，重新安装一次也许就可以解决。