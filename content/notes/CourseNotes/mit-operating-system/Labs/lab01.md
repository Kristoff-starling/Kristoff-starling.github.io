---
title: "MIT-6.S081 Lab 01: Xv6 and Unix utilities"
linktitle: 'Lab util'
type: docs
draft: false
featured: false

math: true

menu:
    mitos-labs:
        parent: Contents
        weight: 1
---

## Progress

- [x] boot xv6 (easy)
- [x] sleep (easy)
- [x] pingpong (easy)
- [x] primes (medium)/(hard)
- [x] find (medium)
- [x] xargs (medium)

## Boot xv6 (easy)

刚刚克隆完项目后文件夹里没有东西，这是正常的。切换到 util 分支之后即可看到内容。

使用 `make grade` 命令进行尝试时发现报错：

```bash
/usr/bin/env: python: No such file or directory
```

笔者根据 stakoverflow 上的[这个回答](https://stackoverflow.com/questions/3655306/ubuntu-usr-bin-env-python-no-such-file-or-directory)进行了修复：首先检查了一下 python3 是否安装，发现已经安装。出现上述报错的原因是系统内只有 python3 这个程序。因此只要创建一个名为 python 的指向 python3 的软链接即可。

```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```

想要使用 gdb 调试 xv6 的话，可以先在一个窗口工作目录下执行

```bash
make CPUS=1 qemu-gdb
```

然后新开一个窗口，在工作目录下执行 `riscv64-linux-gnu-gdb` 或 `gdb-multiarch`，后者会自行判断架构。初次运行可能会遇到 .gdbinit 相关的问题。此时需要创建文件 `~/.gdbinit`，并在其中写

```bash
add-auto-load-safe-path ~/xv6-labs-2020/.gdbinit
```

## sleep (easy)

C 语言编程练习，只需要使用 `sleep()` 系统调用即可。不过需要注意 sleep 的参数规则：

* 如果 sleep 没有参数，应当给出错误提示。
* sleep 可以有多个参数，应该将多个参数相加的值传给 `sleep()` 系统调用。
* sleep 的任何一个参数中有非数字字符时，应当给出错误提示。

## pingpong (easy)

笔者最初只是用了一个管道，在父进程中 `write(p[1], "a", 1)` 后又 `read(p[0], buf, sizeof(buf))`，但这样相当于父进程从管道里读到了自己的输出，没有完成和子进程的互动。

因此需要使用两个管道，第一个管道用于父进程输出，子进程读入，第二个管道用于子进程输出，父进程读入。

## primes (moderate)/(hard)

笔者的初代实现如下：

>  以第一层为例，父进程创建一个管道之后 fork 出一个子进程，然后将 $2,3,\cdots,35$ 喂给管道的输出端。子进程从管道的输入端接收这些数。子进程收到的第一个数一定是质数，它可以用这个数将剩下的数中这个质数的倍数筛除。之后子进程自己变成一个“父进程”，将收到的数中除了最小质数和被筛除的数的其他数传给自己的子进程。该过程一直持续，直到没有数为止。
>
>  核心代码：
>
>  ```c
>  while (tot > 0)
>  {
>   pipe(p);
>  
>   int pid = fork();
>   if (pid < 0) exit_error();
>   else if (pid == 0)
>   {
>       close(p[1]);
>       int cur, curp; tot = -1;
>       while (read(p[0], &cur, sizeof(cur)) > 0)
>       {
>           if (tot == -1)
>           {
>               printf("prime %d\n", cur);
>               tot ++; curp = cur;
>           }
>           else
>           {
>               if (cur % curp != 0) a[tot ++] = cur;
>           }
>       }
>       close(p[0]);
>   }
>   else
>   {
>       close(p[0]);
>       for (int i = 0; i < tot; i ++)
>           write(p[1], a + i, sizeof(int));
>       close(p[1]);
>       int status; wait(&status);
>       if (status != 0) exit_error();
>       exit(0);
>   }
>  }
>  ```

这样做有一个关键的问题：每一层进程都是完全接收了上一层的数，做完筛法i，再把所有的数传给下一层进程，没有使筛法充分并行起来。

真正并发/并行的做法是：每个进程有一个管道和父亲通信，另开一个管道和自己的子进程通信。每从自己的父进程接收到一个数就立刻传给自己的子进程。这样的话，比如在第一层进程中 5 通过了 2 的筛查，被传入了第二层，然后第一层进程在筛查 6，7，8，9... 的时候，第二层进程已经在用 3 筛查 5 了，这样真正做到了并行。将递归与这个过程结合起来写代码会非常舒服，否则需要维护一大堆管道的编号，使编码变复杂。

需要注意的一点是管道的使用。父进程在给子进程传递完数据后，一定要先关闭管道的输出端再调用 `wait()` ，否则子进程的 `read()` 在管道输出没有被全部关闭的情况下是不会退出的。

## find (moderate)

参考 `ls` 程序的实现可以对 `find` 的实现起到很大的帮助。

`find` 的实现需要对 xv6 的文件系统有一个粗浅的了解。以下知识是对该任务的实现有帮助的：

* 当我们拿到一个文件时，可以用 `fstat()` 系统调用来获取它的信息。`fstat()` 系统调用将文件信息写到作为参数的 stat 结构体中，stat 结构体的 type 参数刻画了这是一个文件，文件夹还是设备。

* 在通俗的观念中，文件夹是一个“容器”，里面放了很多的文件。但事实上文件夹本身也是一个文件。按照 xv6 的 `/kernel/fs.h` 中所说，"Directory is a file containing a sequence of dirent structures." dirent 结构体的定义如下：

    ```c
    #define DIRSIZ 14
    
    struct dirent
    {
        int inum;
        char name[DIRSIZ];
    }
    ```

    inum 是该文件占据的 inode 的个数，我们不必访问那些 inum 为 0 的文件。name 是文件名，在 xv6 中，我们默认文件名不超过 14 个字节。因此在 open 了一个文件夹文件后，我们只要不断用 `read(fd, &de, sizeof(dirent))` 命令来读取信息，就可以获得文件夹内所有文件的文件名。将文件名和之前的路径拼接起来，就可以访问文件夹内部的文件。

* 可以用递归来实现文件夹内部文件的访问，但不要对 `.` 和 `..` 递归，否则会造成死递归。

笔者最初实现完发现文件系统后面的若干的文件都显示 `find: cannot open ...`，经过筛查发现这是因为没有多余的文件描述符可以使用了。在 xv6 中，资源比较有限，**我们应该养成打开一个文件，读取完相关信息就关闭它的好习惯**。

## xargs (moderate)

有一些命令不支持从管道读取输入，而 `xargs` 命令可以帮助这些命令从管道读取信息。注意：管道的创建和输出输入的重定向是由 shell 来完成的，我们的 xargs 程序无需关心输入的来源，只需要从标准输入 (文件描述符 1) 读取信息即可。

为新命令准备参数的环节是较好的 C 语言指针练习。笔者新建了一个 `char *child_args[MAXARG]` 数组来为子进程传递参数。xargs 后紧跟的那几个参数是确定的，再后面的参数需要读入，目前还不能确定长度，所以应该在读取完之后用 `malloc()` 来申请内存空间，并在子进程执行完成后用 `free()` 释放空间防止内存泄漏。为了能够准确识别到 `\n` 以执行多条指令，笔者采取了一个字符一个字符地从标准输入读取信息的方式。读取到的字符暂存在局部数组 buf 中，一旦遇到 ' ' 或 '\n' 就根据 buf 的长度申请内存，将 buf 复制到参数数组里。