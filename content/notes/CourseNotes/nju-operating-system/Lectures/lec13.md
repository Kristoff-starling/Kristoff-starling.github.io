---
title: "Lecture 13: System Calls and UNIX Shell"
linktitle: '13: 系统调用和 Shell'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 13
---

> 我们是操作系统用户，但操作系统 API 并不是我们作为人类用户直接使用的，那么“我们”到底应该怎么使用操作系统？

## Shell

我们需要一个与人类交互的程序，这就是 Shell。之所以叫做 shell，是因为它就像包裹住操作系统内核的一层“壳”：它向内使用系统调用与内核交互，向外与人类用户交互。(Shell 不一定是命令行终端！现在的图形界面也是一种 graphical shell。)

Shell 本质上是一门 “把用户指令翻译成系统调用的编程语言”。(在 CLI 时代这一点体现得更明显)

### Xv6 Shell

> 关于 `-ffreestanding`
>
> >  -ffreestanding
> >        Assert that compilation targets a freestanding environment.  This implies -fno-builtin.  A
> >        freestanding environment is one in which the standard library may not exist, and program
> >        startup may not necessarily be at "main".  The most obvious example is an OS kernel.  This is
> >        equivalent to -fno-hosted.
>
> 以上是 man gcc 中关于 `-ffreestanding` 选项的说明。加上这个选项后，编译目标是一个类似于“裸机”的 freestanding 的环境。没有标准库，程序的入口不再是 main() 而是 _start()。

`sh-xv6.c` 是一个零依赖的参考 xv6-riscv 实现的 shell，可以使用 `-ffreestanding` 选项编译到 `.o` 文件并直接用 ld 链接。为了实现零依赖，`sh-xv6.c` 做出了如下的改动：

* `sh-xv6.c` 将 xv6 中的系统调用换成了如下利用 x86-64 syscall 指令的內联汇编代码：

    ```c
    // Minimum runtime library
    long syscall(int num, ...) {
      va_list ap;
      va_start(ap, num);
      register long a0 asm ("rax") = num;
      register long a1 asm ("rdi") = va_arg(ap, long);
      register long a2 asm ("rsi") = va_arg(ap, long);
      register long a3 asm ("rdx") = va_arg(ap, long);
      register long a4 asm ("r10") = va_arg(ap, long);
      va_end(ap);
      asm volatile("syscall"
        : "+r"(a0) : "r"(a1), "r"(a2), "r"(a3), "r"(a4)
        : "memory", "rcx", "r8", "r9", "r11");
      return a0;
    }
    ```

    x86-64 系统调用的各个参数的安放位置可以通过 `man 2 syscall` 查阅。

* `sh-xv6.c` 将 xv6 中所有的 malloc() 替换成了如下的 zalloc()：

    ```c
    static char mem[4096], *freem = mem;
    
    void *zalloc(size_t sz) {
      assert(freem + sz < mem + sizeof(mem));
      void *ret = freem;
      freem += sz;
      return ret;
    }
    ```

    xv6 的写法中 malloc() 没有对应的 free()，这是因为命令总是在 fork 出的子进程上完成的，子进程结束时会自动释放分配的内存。因此 zalloc() 使用了一个 4KB 的数组来模拟堆区，只管分配不管释放，这样 zalloc() 的语义和 malloc() 是相同的 (注：fork() 的时候，父子进程的 mem[] 数组是互不影响的！)。

剩余的部分和 xv6 的 `sh.c` 基本无异，代价解读见 Xv6 源码解读手册。

近距离观测 `sh-xv6.c` 的执行过程，我们有以下两种途径：

* GDB。注意 GDB 默认追踪父进程，我们更希望关注子进程的行为，即指令执行的行为。可以通过 `set follow-fork-mode` `set follow-exec-mode` 等来修改 GDB 的追踪行为。

* strace。为了不将 shell 的输出和 strace 的输出混起来，我们可以这样做：使用命令

    ```bash
    strace -f -o /tmp/strace.log  ./sh
    ```

    将 strace 的输出打印到文件 `/tmp/strace.log` 中，`-f` 选项可以追踪所有子进程的系统调用。然后另开一个窗口执行

    ```bash
    tail -f /tmp/strace.log
    ```

    其中 `-f` 选项会 "output appended data as the file grows" (from `man tail`)。配合 Tmux，我们将获得良好的 strace 观测体验。

### The Shell Programming Language

使用 shell 本质上就是在编程。我们组合各种指令来完成复杂的任务。shell 提供基于文本替换的快速工作流的搭建：

* 重定向：`cmd > /dev/null`
* 顺序结构：`cmd1 ; cmd2` (两者都执行)，`cmd1 && cmd2` (前者返回值为 0 才继续执行后者)，`cmd1 || cmd2` (前者返回值为 1 才继续执行后者) etc.
* 管道：`cmd1 | cmd2`
* 预处理：`$()`  (将一个命令的输出作为另一个命令的参数)，`<()` (将一个命令的输出重定向到一个文件，返回文件描述符作为另一个命令的参数) etc.
* 环境变量

现代 GUI 能完成的事情绝大多数 CLI 也能完成，比如在图形界面中，我们通过点"叉" “最小化” 来管理前后台任务，窗口。在 CLI 中我们也有 job control。在执行命令时，我们可以在后面加上 `&` 使其在后台执行；对于一个正在前台运行的程序，我们可以通过 `ctrl+z` 将其暂时挂起，用 `bg %num` 将指定进程放到后台执行；对于在后台运行的程序，我们可以用 `fg %num` 将其拉到前台执行。

---

UNIX shell 在自然语言、机器语言和1970s算力之间达到了一个优雅的平衡。但平衡意味着 UNIX shell 的设计也不是完美的。例如操作的优先级：

```bash
ls > a.txt | cat
```

重定向和管道的优先级谁高？使用不同的 shell会得到不同的结果。(bash 不会打印到终端，zsh 会打印到终端 etc.)

再例如文件名中的空格会带来很大的危险：如果使用命令 `vim "a b.txt"`，事实上我们连续打开了 `a` 和 `b.txt`，这与我们的预期不同。这本质上是因为空格具有二义性：它有可能是分隔指令的符号。(Windows 的 powershell 有 object stream pipe，在管道中传递的东西不再是文本而是对象，这使得所有的东西都是 strongly typed 的，可以避免上述问题。)

再比如，有些行为的意义可能是难以理解的：

```bash
echo hello > /etc/a.txt
```

无法执行，shell 会返回 permission denied。这时我们本能地在前面加上 sudo：

```bash
sudo echo hello > /etc/a.txt
```

却发现仍然是 permission denied。联想 xv6-shell 中解析命令的机制，我们就不难理解这个问题的原因。shell 完成重定向输出的方法是：关闭 stdout 文件，打开 `/etc/a.txt` 文件 (其文件描述符绑定为 1)，然后执行左边的指令。也就是说，sudo 只修饰了左边的执行，打开文件的过程没有获得高权限，所以报错。

## TTY and Job Control

我们还有一些疑问：例如没有任何程序在读键盘输入的情况下为什么 Ctrl+C 可以退出程序？为什么在有些应用 (e.g. vim) 中 Ctrl+C 不能退出？如果当前程序 fork 出了多个进程，Ctrl+C 会退出所有程序吗？Tmux 为什么能管理多个窗口？

答案是终端。终端是 UNIX 操作系统中的一类非常特别的设备。命令行中有工具 tty 可以查看连接到标准输入的终端文件名。如果我们在 Tmux 的不同窗口中使用 tty 命令，可以看到不同的窗口连接到了不同的终端设备。我们甚至可以做一些有趣的事情：在一个窗口中 `echo hello > /dev/pts/other`，hello 将显示在另一个窗口中。

> 为什么 `fork-printf.c` 可以根据标准输出对象的不同选择不同的 buffer mode？

通过 strace 观察，可以看到程序使用了 fstat() 系统调用查看 1 号文件 (标准输出) 的信息。在有重定向的版本的系统调用序列中，ioctl() 系统调用返回了 ENOTTY。

### Session, Process Group and Signal

> Ctrl+C 为什么能退出？

简单来说，我们的终端负责识别 Ctrl+C 的按键组合，并给**前台**进程发送一个 SIGINT 信号。前台程序有自由决定如何处理这个 SIGINT 信号，绝大部分的程序会在接收到 SIGINT 信号后退出，但我们也可以自己写一个 signal handler：

```c
// signal-handler.c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void handler(int signum) {
  switch (signum) {
    case SIGINT:
      printf("Received SIGINT!\n");
      break;
    case SIGQUIT:
      printf("Received SIGQUIT!\n");
      exit(0);
      break;
  }
}

void cleanup() {
  printf("atexit() cleanup\n");
}

int main() {
  signal(SIGINT,  handler);
  signal(SIGQUIT, handler);
  atexit(cleanup);

  while (1) {
    char buf[4096];
    int nread = read(STDIN_FILENO, buf, sizeof(buf));
    buf[nread - 1] = '\0';
    printf("[%d] Got: %s\n", getpid(), buf);
    if (nread < 0) {
      perror("read");
      exit(1);
    }
    sleep(1);
  }
}
```

 上面的程序为 SIGINT 和 SIGQUIT 两个信号注册了处理函数。运行该程序，我们发现按下 Ctrl+C 时程序不会退出，而是输出 `Received SIGINT!`。

这时有一个更有意思的问题：如果我们在 main() 函数开头执行一个 fork()，那么父子进程的标准输入和输出会连向同一个 tty，这时按下 Ctrl+C 终端会将 SIGINT 发送给哪个进程呢？经过实验我们发现：fork() 以后的 `signal-handler.c` 在输入字符时，两个进程会争抢输入数据，呈现各打印了一部分内容的现象；如果按下 Ctrl+C，会出现两个 `Received SIGINT!`。这说明 SIGINT 信号同时发送给了这两个进程。

阅读 setpgid() 的手册，我们可以看到更系统的解读：

当我们打开一个 shell 时，我们就创建了一个会话 (session)，这个会话有一个 controlling terminal。一个会话里面可以有若干个进程组 (process group) (注：shell 本身也是一个进程组，通常称为 session leader)，一个进程 fork() 得到的子进程和父进程同属一个进程组，execve() 不会改变一个进程所属的进程组。在任何时刻，一个会话里有且仅有一个前台 (foreground) 进程组，剩余的进程组都是后台进程组。当终端给会话发送一个信号时，前台进程组里的每个进程都会接收到这个信号。只有前台进程组中的进程可以从终端读取数据，如果后台进程尝试从终端读取数据，会接收到一个 SIGTTIN 信号，该信号会将后台进程挂起。