---
title: "Lecture 11: Processes in Operating Systems"
linktitle: '11: OS的进程'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 11
---

在 `thread-os.c` 中，操作系统启动后会初始化若干个线程 (的状态机)，或者说加载若干个程序。真正的操作系统启动后会创建第一个进程。如果查看 Linux 的源码，我们可以看到：

```c
if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;
panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/admin-guide/init.rst for guidance.");
```

内核会按照一个特定的顺序加载“第一个程序”，如果都无法加载，Linux kernel 会直接拒绝启动。

## A Minimal Linux

我们可以在 qemu 上调试 vmlinuz 内核。如下命令会启动一个没有图形界面，128M 内存的 linux 内核：

```bash
qemu-system-x86_64 \
	  -nographic \
	  -serial mon:stdio \
	  -m 128 \
	  -kernel vmlinuz \
	  -initrd build/initramfs.cpio.gz \
	  -append "console=ttyS0 quiet acpi=off"
```

其中 `initramfs.cpio.gz` 是内核启动时存放在内存中的一个很小的文件系统的镜像，它打包了如下目录结构中地 initramfs 中的内容。

```
├── initramfs
│   ├── bin
│   │   └── busybox
│   └── init
├── Makefile
└── vmlinuz
```

启动内核后我们会得到一个 shell，当前的环境很简陋，但我们仍然可以用 `/bin/busybox 命令` 的方式使用一些 busybox 内置的命令，比如 ls 和 cat。

操作系统启动的第一个进程是 init。init 是一个脚本，有一行命令：`/bin/busybox sh`，它启动一个 shell。我们可以试试将 `/bin/busybox sh` 换成 `/bin/busybox echo hello`：可以看到操作系统确实打印了 hello，但随即报了 kernel panic。这是因为初始进程 init 返回了。

尽管现在的环境很简陋，但我们执行一些简单的代码就可以获得丰富的体验：

```bash
c1="arch ash base64 cat chattr chgrp chmod chown conspy cp cpio cttyhack date dd df dmesg dnsdomainname dumpkmap echo ed egrep false fatattr fdflush fgrep fsync getopt grep gunzip gzip hostname hush ionice iostat ipcalc kbd_mode kill link linux32 linux64 ln login ls lsattr lzop makemime mkdir mknod mktemp more mount mountpoint mpstat mt mv netstat nice nuke pidof ping ping6 pipe_progress printenv ps pwd reformime resume rev rm rmdir rpm run-parts scriptreplay sed setarch setpriv setserial sh sleep stat stty su sync tar touch true umount uname usleep vi watch zcat"
c2="[ [[ awk basename bc beep blkdiscard bunzip2 bzcat bzip2 cal chpst chrt chvt cksum clear cmp comm crontab cryptpw cut dc deallocvt diff dirname dos2unix dpkg dpkg-deb du dumpleases eject env envdir envuidgid expand expr factor fallocate fgconsole find flock fold free ftpget ftpput fuser groups hd head hexdump hexedit hostid id install ipcrm ipcs killall last less logger logname lpq lpr lsof lspci lsscsi lsusb lzcat lzma man md5sum mesg microcom mkfifo mkpasswd nc nl nmeter nohup nproc nsenter nslookup od openvt passwd paste patch pgrep pkill pmap printf pscan"
c3="pstree pwdx readlink realpath renice reset resize rpm2cpio runsv runsvdir rx script seq setfattr setkeycodes setsid setuidgid sha1sum sha256sum sha3sum sha512sum showkey shred shuf smemcap softlimit sort split ssl_client strings sum sv svc svok tac tail taskset tcpsvd tee telnet test tftp time timeout top tr traceroute traceroute6 truncate ts tty ttysize udhcpc6 udpsvd unexpand uniq unix2dos unlink unlzma unshare unxz unzip uptime users uudecode uuencode vlock volname w wall wc wget which who whoami whois xargs xxd xz xzcat yes"
for cmd in $c1 $c2 $c3; do
  /bin/busybox ln -s /bin/busybox /bin/$cmd
done
mkdir -p /proc && mount -t proc  none /proc
mkdir -p /sys  && mount -t sysfs none /sys
export PS1='(linux) '
```

现在我们可以直接使用命令不再需要加 /bin/busybox 的前缀 (得益于代码中的软链接)，我们还挂载了 procfs 和 sysfs。可以使用 ps 等命令查看进程。此外，我们可以运行任何放入到文件系统中的程序：例如之前的示例代码 `minimal.S` 和 `logisim.c`，只要是静态链接得到的可执行文件都可以在我们的 minimal Linux 中直接运行。

这一切说明操作系统没有什么神秘的：**内核创建了第一个进程 init，然后这个进程通过各种各样的系统调用就能创造全世界。**完成一切以后操作系统将退到幕后成为一个“中断处理程序”。此外，操作系统为应用程序提供各类 API，让应用程序创建/管理进程，地址空间，文件系统等。

## fork()

fork() 的含义非常简单：它会将当前进程的状态机完全复制一遍，作为当前进程的子进程。这两个进程几乎完全一样：相同的地址空间，相同的寄存器值…… (当然，进程号是不同的)，除了 fork() 的返回值在两个进程中不同：fork() 在父进程中返回子进程的进程号，在子进程中返回 0。

> **Fork Bomb**
>
> 状态机的复制也需要消耗一定的资源。如下的一段代码可以让 OS 崩溃：
>
> ```bash
> :() { :|:& };:
> ```
>
> 将其格式化后可以看出它的原理：
>
> ```bash
> :() {
>  : | : &
> }; :
> ```
>
> bash 允许冒号作为标识符。: 这个函数的作用是用 fork 创建两个自己，用管道连接起来，然后放到后台执行。这样这个脚本就会像链式反应一样不断生成很多函数，直至系统资源耗尽。

### Example: `fork-demo.c`

```c
// fork-demo.c
#include <unistd.h>
#include <stdio.h>

int main() {
  pid_t pid1 = fork();
  pid_t pid2 = fork();
  pid_t pid3 = fork();
  printf("Hello World from (%d, %d, %d)\n", pid1, pid2, pid3);
}
```

运行这段程序，我们可以看到所有的 8 种可能的结果：进程号是 0 / 真实值。我们可以画出状态机中的一条路径来理解这个输出 (即不考虑多个进程时程序的 non-deterministic 行为，总是挑选可以继续执行的进程号最小的进程走下一步)：

![pic](/img/njuos-lec11.png)

### Example: `fork-printf.c`

```c
// fork-printf.c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
  int n = 2;
  for (int i = 0; i < n; i++) {
    fork();
    printf("Hello\n");
  }
  for (int i = 0; i < n; i++) {
    wait(NULL);
  }
}
```

从抽象的状态机的角度分析，刚开始只有一个进程，第一轮一个进程变成两个进程，并分别打印 hello；第二轮两个进程变成 4 个进程，并分别打印 hello，最后应该总共有 6 个 hello。

运行该程序可以看到，终端上确实输出了 6 个 hello，但诡异的事情是：如果我们用管道或者重定向到文件的方式，都会得到 8 个 hello。

```bash
./fork-printf | cat    # 8个hello
./fork-printf > output # 8个hello
./fork-printf | wc -l  # 8
```

这件“诡异”的事情和 printf 的缓冲区有关。我们认为 printf 的语义是立即输出给定的字符串，但 printf 为了性能考虑 (尽可能少地与 io 交互)，会设置 buffer，将要输出的内容先预留在 buffer 中，待 buffer 满了 / 进程结束时再将 buffer 中的内容一并输出。

printf 根据标准输出指向的文件不同会使用不同的 buffer。当 stdout 是 tty 时，printf 使用 line buffer，即遇到换行符会清空一次 buffer，所以当我们输出到终端时看到的结果是 6 个；当 stdout 是管道或者文件时，printf 使用 full buffer，只有 buffer 满了才会清空。在这种情况下，第一轮的两个进程不会输出 hello，而是会将 hello 保存在 buffer 中，第二轮两个进程变 4 个进程时，fork() 复制状态机，当然也会复制 buffer 里的内容，因此新生成的两个进程的 buffer 里自带一个 hello，再执行 printf 后，每个进程的 buffer 里都有两个 hello。进程结束时内核清空 buffer，于是 4 个进程每个打印两个 hello，最终有 8 个 hello。

如果我们在程序开头加一句 `setbuf(stdout, NULL)`，指定 stdout 不使用 buffer，或者在 printf() 之后加 `ffush(stdout)` ，那么各种情况下我们都会得到 6 个 hello。

> **Tip: setbuf()**
>
> setbuf() 函数可以指定向某个文件输出时使用的 buffer。与之相关的还有 setvbuf() （可以用 `_IONBF`  `_IOLBF` `_IOFBF` 制定无缓冲、行缓冲、全缓冲）等。
>
> 使用 setbuf() 时要小心的一点是我们注册的 buffer 不能在输出流被关闭之前被释放，例如下面的程序就是错误的 (undefined behavior)：
>
> ```c
> int main ()
> {
>     char buf[64];
>     setbuf(stdout, buf);
>     printf("Hello, world\n");
>     return 0;
> }
> ```

**计算机系统里没有魔法。机器永远是对的。**

## execve()

execve() 的作用是重置一个状态机，运行 execve() 的参数中所写的程序。如果该系统调用执行成功，它不会返回，它后面的指令也不会被执行。结合运用 fork() 和 execve()，我们便可以创建新的状态机并运行新的程序。execve() 的声明如下：

```c
int execve(const char *filename, char * const argv, char * const envp);
```

其中 filename 为想要加载的程序，argv 为传给新程序入口函数的参数，envp 为环境变量列表。下面是一个简单的示例程序：

```c
// execve-demo.c
#include <unistd.h>
#include <stdio.h>

int main() {
  char * const argv[] = {
    "/bin/bash", "-c", "env", NULL,
  };
  char * const envp[] = {
    "HELLO=WORLD", NULL,
  };
  execve(argv[0], argv, envp);
  printf("Should not reach here!\n");
}
```

它使用 execve() 执行了命令 `/bin/bash -c env` 来打印环境变量。运行该程序，我们可以看到我们指定的环境变量 HELLO=WORLD 被打印，且 "should not reach here" 没有被打印。

系统中有很多有意思的环境变量。例如我们可以配置 `$AM_HOME` 来简写目录；bash 中的环境变量 PS1 决定了命令行提示符长什么样子；execvp()/gcc 等会根据 `$PATH` 中的目录一个一个去找有没有我们指定的程序 etc。我们可以 hack 这个行为：

```bash
PATH= /bin/gcc program.c
```

便会看到

```bash
gcc: fatal error: cannot execute ‘as’: execvp: No such file or directory
compilation terminated.
```

## _exit()

exit() 用于销毁一个状态机。`_exit(int status)` 会销毁当前状态机，并允许有一个返回值。子进程被销毁了会通知父进程。这其中有一些值得思考的问题，例如如果这是一个多线程程序，exit() 应当销毁当前的线程还是当前的进程？

```c
// exit-demo.c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <sys/syscall.h>

void func() {
  printf("Goodbye, Cruel OS World!\n");
}

int main(int argc, char *argv[]) {
  atexit(func);
  setvbuf(stdout, NULL, _IOFBF, 1024);
  printf("Unflushed Buffer\n");

  if (argc < 2) return EXIT_FAILURE;

  if (strcmp(argv[1], "exit") == 0)
    exit(0);
  if (strcmp(argv[1], "_exit") == 0)
    _exit(0);
  if (strcmp(argv[1], "__exit") == 0)
    syscall(SYS_exit, 0);
}
```

`exit-demo.c` 展示了各种不同的 exit 方法行为的不同，其中 atexit() 注册了一个在 exit() 的时候会调用的函数。各种运行方式的结果如下：

| 命令                 | 执行内容               | 系统调用      | 是否调用 atexit | 输出内容      |
| -------------------- | ---------------------- | ------------- | --------------- | ------------- |
| `./exit-demo`        | `return EXIT_FAILURE`  | exit_group(1) | 是              | func & buffer |
| `./exit-demo exit`   | `exit(0)`              | exit_group(0) | 是              | func & buffer |
| `./exit-demo _exit`  | `_exit(0)`             | exit_group(0) | 否              | 无            |
| `./exit-demo __exit` | `syscall(SYS_exit, 0)` | exit(0)       | 否              | 无            |

注：C 程序中的 exit(0) 是 `stdlib.h` 中的 libc 函数。

注：exit_group() 系统调用会终止当前进程的所有线程，而 exit() 系统调用只终止当前线程。