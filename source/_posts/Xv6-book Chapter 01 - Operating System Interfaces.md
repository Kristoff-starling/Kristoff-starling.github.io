---
layout: post
date: 2022-01-20
title: "Xv6-book Chapter 01: Operating System Interfaces"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

## 1.1 Processes and memory

`fork()` 系统调用可以创建一个和当前进程存储内容完全相同的子进程（原进程称为父进程）。`fork()` 在父进程和子进程中都会返回，在父进程中返回创建的子进程的进程号，在子进程中返回 0。如果返回 -1 说明发生错误。

<!-- more -->

```c
int pid = fork();
if (pid > 0)
{
    printf("parent: child=%d\n", pid);
    pid = wait((int *)0);
    printf("child %d is done\n", pid);
}
else if (pid == 0)
{
    printf("child: existing\n");
    exit(0);
}
else
{
    printf("fork error\n");
}
```

`fork()` 系统调用使用后，父进程和子进程会同时运行（不过两个进程中的 pid 变量的值不同）。

* 父进程进入 if 的第一个分支，打印信息后使用 `wait()` 系统调用。`wait(int *status)` 系统调用会返回退出的/被杀死的子进程的进程号，并将该子进程的状态写到 status 中。如果当前进程没有子进程 `wait()` 直接返回 -1；如果当前进程的子进程都没有退出/被杀死，`wait()` 会等待直到这件事发生。
* 子进程进入 if 的第二个分支，打印信息后使用 `exit(int status)` 系统调用返回。status 为 0 说明正常退出，status 为 1 说明非正常退出。

该程序的输出结果为

```c
parent: child=1234
child: existing			// 这两行的先后顺序不确定，取决于哪个进程的 printf 先被执行
child 1234 is done		// 子进程结束后，父进程会输出这句话
```

---

Xv6 的 shell 使用 `exec()` 系统调用来加载程序。如果成功加载，`exec()` 调用不会返回。`sh.c` 的关键代码如下：

```c
while (getcmd(buf, sizeof(buf)) >= 0)
{
    ...
    if (fork1() == 0)			// fork1()是对fork()系统调用的进一步封装，检查了返回值
        runcmd(parsecmd(buf));
   	wait(0)
}
```

一旦有新的输入内容到来，shell 进程会 fork 出一个子进程，在子进程中运行程序，父进程 wait 直到子进程运行完成。`parsecmd()` 负责解析输入内容并打包成一个 cmd 结构体，`runcmd()` 负责运行一个 cmd 结构体，正常情况下不会返回：

```c
void runcmd(struct cmd *cmd)
{
    ...
	switch(cmd->type)
    {
        default: panic("runcmd");	// panic()会向标准错误流输出信息，并exit(1)

        case EXEC:
            ecmd = (struct execcmd*)cmd;
            if (ecmd->argv[0] == 0) exit(1);
            exec(ecmd->argv[0], ecmd->argv);
            fprintf(2, "exec %s faile\n", ecmd->argv[0]);
            break;
        ...
    }
}
```

通常来说 argv[0] 默认为文件名。如果 `exec()` 系统调用返回了，说明发生了错误，于是向标准错误流输出错误信息。(注：`exec()` 系统调用不会创建新进程，只会清空当前进程的 memory。)

## 1.2 I/O and File descriptors

内核为每个进程都记录了一张表格，表格维护了每一个文件描述符对应的文件名。默认情况下，应用程序从 0 号文件读入（标准输入），1 号文件输出（标准输出），并将错误信息输出到 2 号文件（标准错误）。使用 `open()` 系统调用打开文件时，内核总是会把当前进程中的第一个未被分配的文件描述符索引到这个文件上。

将 `fork()` 和 `open()` 系统调用配合使用可以实现输入和输出的重定向，以 `cat < input.txt` 为例：

```c
char *argv[2];

argv[0] = "cat";
argv[1] = NULL;
if (fork() == 0)
{
    close(0);
    open("input.txt", O_RDONLY); 、
        // 第二个参数描述文件的权限，有O_RDONLY,O_WRONLY,O_RDWR,O_CREATE,O_TRUNC等
    exec("cat", argv);
}
```

使用 `fork()` 创建子进程时子进程复制了父进程的所有文件描述符，在子进程中关闭控制台输入，并将 0 号文件重新绑定到 input.txt 上，从而完成了输入重定向。子进程中进行的这些操作是不会影响父进程的。

虽然 `fork()` 克隆的子进程是复制了一遍父进程的文件描述符，但子进程和父进程的文件共享文件偏移。比如

```c
if (fork() == 0)
{
    write(1, "Hello ", 6);
    exit(0);
}
else
{
    wait(0);
    write(1, "World\n", 6);
}
```

将会输出 "Hello World\n"，这是父进程和子进程共享 1 号文件的偏移的结果。

使用 `dup()` 系统调用可以复制一个文件描述符，它会返回一个新的文件描述符，新的文件描述符和被复制的指向同一个文件，且共享文件偏移。比如

```c
int fd = dup(1);
write(1, "Hello ", 6);
write(fd, "World\n", 6);
```

将会输出 "Hello World\n"。**`fork()` 和 `dup()` 是唯二可以使得两个文件描述符共享文件偏移的系统调用。**我们平时在命令行中使用的 `2>&1`，其原理就是 `dup()` 系统调用。它的功能类似于

```c
close(2);
int fd = dup(1);	// 这时2号是第一个未被分配的文件描述符
```

## 1.3 Pipes

`pipe()` 的声明为

```c
int pipe(int pipefd[2]);
```

`pipe()` 创建一个管道，在传进去的数组中填写两个文件描述符，pipefd[0] 是管道的输入端，pipefd[1] 是管道的输出端。`pipe()` 的返回值用于检验管道创建是否成功。

下面是 xv6 的 `sh.c` 中使用管道的方法：

```c
void runcmd(struct cmd* cmd)
{
    ...
    switch(cmd->type)
    {
    	...
        case PIPE:
            int p[2];
    		pcmd = (struct pipecmd*)cmd;
            if (pipe(p) < 0) panic("pipe");
            if (fork1() == 0)
            {
                close(1);
                dup(p[1]);
                close(p[0]);
                close{p[1]);
                runcmd(pcmd->left);
            }
            if (fork1() == 0)
            {
                close(0);
                dup(p[0]);
                close(p[0]);
                close(p[1]);
                runcmd(pcmd->right);
            }
            close(p[0]);
            close(p[1]);
            wait(0);
            wait(0);
            break;
    }
}
```

进入 `runcmd()` 的是执行当前命令的 shell 进程的子进程。该子进程在创建管道之后又创建了两个子进程，一个子进程将标准输出重定向到管道的输出端，并执行 "|" 左边的命令；另一个子进程将标准输入重定向到管道的输入端，并执行 "|" 右边的指令（注：这两个子进程互不影响）。考虑到命令可能会有多级管道（如 `a | b | c` 的形式），进程之间会形成一个树形结构，树的叶子节点在真正地执行指令，其他节点都在负责创建管道和子进程。

该代码有几个值得注意的细节：

* **通过管道读取的 `read()` 系统调用在没有数据输入的时候会被阻塞，直到新的数据到来或者和管道输出端相连的所有文件描述符均被关闭**（在管道输出端已经没有打开的文件描述符时，`read()` 会返回 0）。因此父进程必须在两个 if 块后面将和管道输出绑定的 p[1] 关闭，第二个 if 块克隆的用于输入的子进程也必须将和管道输出绑定的 p[1] 关闭，否则 `read()` 将陷入死循环。下面的例子展示了忽略某些管道输出的关闭会造成死循环的后果：

    ```c
    int p[2];
    pipe(p);
    if (fork() == 0)
    {
        close(0);
        dup(p[0]);      // 将标准输入重定向到管道输入
        close(p[0]);
        close(p[1]);    // ***
        int buf[64];
        while (n = read(0, buf, sizeof(buf)) > 0) write(1, buf, n);
        exit(0);
    }
    if (fork() == 0)
    {
        close(1);
        dup(p[1]);       // 将标准输出重定向到管道输出
        close(p[0]);
        close(p[1]);
        write(1, "Hello\n", 6);
        exit(0);
    }
    close(p[0]);
    close(p[1]);         // ***
    wait(0);
    wait(0);
    ```

    标有 `***` 的两处管道输出关闭，去掉任何一个都会导致第一个子进程中的 `read()` 系统调用无法正确识别输入的结束，陷入死循环。

* 每使用一次 `wait()` 系统调用，父进程都会等待直到某一个子进程终止。该程序中父进程 fork 了两个子进程，因此需要连续执行两次 `wait()` 系统调用。

## 1.4 File systems

以 `/` 开头的路径是绝对路径，否则是相对路径。相对路径会在当前工作目录下寻找文件。`chdir()` 系统调用可以修改当前的工作目录。下面的例子中，两种操作做的事情等价：

```c
chdir("/a");
chdir("b");
open("c", O_RDONLY);

open("/a/b/c", O_RDONLY);
```

`cd` 命令是一个特殊的命令，它的作用是切换当前的工作目录，其特殊之处在于我们不能 fork 一个子进程并执行这个程序，否则只有子进程的工作目录被修改，子进程退出之后父进程的工作目录没有变。`cd` 命令直接在原进程中使用。

创建文件和文件夹的系统调用有 `open()` `mkdir()` `mknod()` 等。`open()` 的第二个参数中或上 O_CREATE 可以在文件不存在时创建它；`mkdir()` 用于创建文件夹；`mknod()` 可以创建一个设备文件。

---

文件名和其对应的文件是两个概念。每个文件有一个唯一的文件索引号 (inode)，文件可以有若干个文件名，这些文件名都指向该文件的 inode。使用 `fstat()` 系统调用可以查看一个文件的 inode 信息。`fstat()` 的声明为：

```c
int fstat(int fd, struct stat* stat);
```

stat 结构体的格式如下：

```c
#define T_DIR		1
#define T_FILE		2
#define T_DEVICE	3

struct stat
{
    int dev;        // 文件系统的磁盘设备
    uint ino;       // inode号
    short type;     // 文件类型，上方三个宏中的一个
    short nlink;    // 指向该inode的文件名的个数
    uint64 size;    // 文件大小
};
```

`link()` 系统调用可以将一个新的文件名指向某一个老的文件名指向的 inode，`unlink()` 系统调用可以将文件名和 inode 的关联删除。例如

```c
link("a", "b");
unlink("a");
```

第一条语句过后，使用文件名 "a" 和 "b" 将访问到同一个文件。第二条语句过后，只能使用文件名 "b" 访问该文件

## 1.5 Real world

略。