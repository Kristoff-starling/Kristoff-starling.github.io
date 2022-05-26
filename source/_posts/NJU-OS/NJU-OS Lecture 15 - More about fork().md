---
layout: post
date: 2022-04-09
title: "NJU-OS Lecture 15: More about fork()"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## `fork()` Semantics

fork() 的语义是将当前进程的状态机完整地复制一份，这两个进程除了 fork() 的返回值以外全部相同。

### File Descriptors

我们这里关注文件描述符：根据 fork() 的语义，子进程和父进程会拥有相同的文件描述符 (文件描述符可以理解为指向操作系统对象的指针)。execve() 会重置状态机，但会继承原进程持有的所有操作系统对象。这个设计使我们在 UNIX shell 中可以利用管道技术，在 execve() 之前修改 stdin, stdout 以完成进程之间数据的传输。<!-- more -->

> **execve() 会无条件继承所有的对象吗？**
>
> 在 open() 的手册中，我们可以看到打开文件时有一个标志是 O_CLOEXEC，这样打开的文件在 execve() 的时候不会被继承。

文件描述符其实不仅仅是指向对象的指针。我们在一个程序中写两句话

```c
write(fd, "Hello", 5);
write(fd, "World", 5);
```

我们得到的会是 "HelloWorld" 而不是 "World"，这说明文件描述符中还保存了一个对象当前的 offset。那么一个自然的问题是：如果我们 fork 出一个子线程，然后父子线程分别打印 Hello 和 World，我们会得到两个字符串还是一个字符串呢？我们可以写一个程序验证一下：

```c
// fork-fd.c
#include <fcntl.h>
#include <unistd.h>
#include <wait.h>

int main ()
{
	int fd = open("a.txt", O_RDWR | O_CREAT | O_TRUNC);
	int rt = fork();
	if (rt == 0)
		write(fd, "World", 5);
	else
		write(fd, "Hello", 5);
	wait(NULL);
	if (rt == 0) write(fd, "\n", 1);
	return 0;
}
```

该程序会输出 "HelloWorld" (或者 "WorldHello")，而不是只有五个字符。这说明 fork 之后的两个进程后续仍然是共享文件偏移的。这样的设计显然符合一个“正常“的程序员的需求。查阅 dup() 系统调用的手册，我们可以看到 dup() 获得的新文件描述符和被复制的文件描述符也是共享文件偏移的。

共享偏移对操作系统内核提出了较强的挑战。内核必须非常小心，保证对文件描述符的操作是原子的 (在 `man 2 write` 中可以看到 Linux 内核关于 file offset 的 bug 在内核 3.14 版本中才得到修复)。

### Copy-on-write Fork

fork() 的语义是完整复制了状态机，但在实现层面，现代内核并不是立即复制所有的数据。fork() 有大量的场景都是为了启动一个新程序，即 fork() + execve() 的组合，这种情况下我们复制完的地址空间会立刻被新的程序镜像覆盖，复制工作就很浪费。

现代内核普遍使用 copy-on-write fork (写时拷贝) 技术，即 fork() 时子进程完整复制父进程的页表，两者指向相同的页面，并暂时将页面的写权限去掉。这样之后如果父子进程读取数据，两者可以并行不悖。如果某个进程需要写数据，会因为没有写权限触发 page fault，内核的 page fault handler 发现这是一个 cow 页面，便会现场复制一个新的页面，让父子进程指向不同的页面，并把写权限恢复。

cow fork 在实现层面还有一些细节，比如每个页面需要维护一个 reference count，之后对一个页面 malloc() 和 free() 的语义要做出一些修改，以及根据 ISA 确定如何在页表项中添加 cow 页面标志位等等。

我们可以通过一个小程序感受 Linux 内核的 copy-on-write fork：

```c
// cow-test.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <assert.h>
#include <string.h>

#define NPROC 1000
#define MB 128
#define SIZE (MB * (1 << 20))

#define xstr(s) str(s)
#define str(s) #s

int main() {
  char *data = malloc(SIZE); // 128MB shared memory
  memset(data, '_', SIZE);

  for (int i = 0; i < NPROC - 1; i++) {
    if (fork() == 0) break;
  }

  // NPROC processes go here

  asm volatile(".fill 1048576 * " xstr(MB) ", 1, 0x90"); // 128MB shared code

  unsigned int idx = 0;
  int fd = open("/dev/urandom", O_RDONLY); assert(fd > 0);
  read(fd, &idx, sizeof(idx));
  close(fd);
  idx %= 1048576 * MB;

  data[idx] = '.';
  printf("pid = %d, write data[%u]\n", getpid(), idx);

  while (1) {
    sleep(1); // not terminate
  }
}
```

在 `cow-test.c` 中，我们用 fork() 创建了 1000 个子进程，每个进程都有一个大小为 128M 的代码区 (都是 nop 指令)，每个进程还会随机挑选一个位置进行修改。编译后，我们的可执行文件大小就在 128M 左右。如果内核使用普通的 fork，这个程序运行到一半就会因内存不足而崩溃。但事实上 Linux 运行的很好。这证明了 Linux 中使用了 cow fork 策略。

cow fork 策略的一个推论便是：想要统计一个程序运行时占用的内存空间是一个伪命题 (比如操作系统里的 libc 代码只有一份，不知道有多少程序的页表指向了它)。

## State Machine, `fork()` and Magic

fork() 可以完整复制状态机，这可以帮助我们完成很多炫酷的事情，比如开平行宇宙做 non-deterministic 的事情。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdint.h>
#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/wait.h>

#define DEST  '+'
#define EMPTY '.'

struct move {
  int x, y, ch;
} moves[] = {
  { 0, 1, '>' },
  { 1, 0, 'v' },
  { 0, -1, '<' },
  { -1, 0, '^' },
};

char map[][512] = {
  "#######",
  "#.#.#+#",
  "#.....#",
  "#.....#",
  "#...#.#",
  "#######",
  "",
};

void display();

void dfs(int x, int y) {
  if (map[x][y] == DEST) {
    display();
  } else {
    int nfork = 0;

    for (struct move *m = moves; m < moves + 4; m++) {
      int x1 = x + m->x, y1 = y + m->y;
      if (map[x1][y1] == DEST || map[x1][y1] == EMPTY) {
        int pid = fork(); assert(pid >= 0);
        if (pid == 0) { // map[][] copied
          map[x][y] = m->ch;
          dfs(x1, y1);
          exit(0); // clobbered map[][] discarded
        } else {
          nfork++;
          waitpid(pid, NULL, 0); // wait here to serialize the search
        }
      }
    }

    while (nfork--) wait(NULL);
  }
}

int main() {
  dfs(1, 1);
}

void display() {
  for (int i = 0; ; i++) {
    for (const char *s = map[i]; *s; s++) {
      switch (*s) {
        case EMPTY: printf("   "); break;
        case DEST : printf(" ○ "); break;
        case '>'  : printf(" → "); break;
        case '<'  : printf(" ← "); break;
        case '^'  : printf(" ↑ "); break;
        case 'v'  : printf(" ↓ "); break;
        default   : printf("▇▇▇"); break;
      }
    }
    printf("\n");
    if (strlen(map[i]) == 0) break;
  }
  fflush(stdout);
  sleep(1); // to see the effect of parallel search
}
```

`dfs-fork.c` 是一个走迷宫程序，不同于基本的 dfs 走迷宫，它的 dfs() 函数中每发现一个新的路，便会 fork 一个新的进程去做它，结尾处的 `sleep(1)` 用于模拟一个耗时很长的任务。现在的 `dfs-fork.c` 还不能做到并行，因为 dfs 函数中当前进程会 wait 一个分支的子进程结束才会探索下一个分支。但我们如果把 dfs 循环内的 wait 去掉，就可以观测到程序“秒”完成任务。

fork() 相较于搜索-回溯还有一个好处：当我们探索一个分支失败时，我们不用一步一步撤回去，而是可以直接销毁当前进程，然后从之前保存的副本出发搜索新的分支。

### Parallel Universe: Skipping Initialization

以 NEMU 举例，我们可能需要运行很多个 benchmark，但 NEMU 的初始化过程很长，我们能不能只进行一次初始化，然后让多个 benchmark 共享这一次初始化呢？

```c
int main() {
    nemu_init();
    while (1) {
        file = get_start_request();
        if ((pid = fork()) == 0) {
            load_file();
            ...
        }
    }
}
```

上面一段 C 代码大致实现了我们的想法。我们在 init() 结束后，每次 fork 一个子进程做一个 benchmark，就可以让多个 benchmark 共享初始化刚结束时的状态机了。

类似的思想在实际中也有广泛应用：

* Zygote Process：zygote 是受精卵的意思，这是 Android 里的一个机制。Java 程序启动需要加载很多的 class node，这个过程很慢，但事实上现在的 Android app 基本都是秒起的。这是因为 Android 的一个 zygote process 完成所有的初始化工作，然后每开一个 app 直接在 zygote process 的基础上 fork，设置一些权限、用户 id 之后很快就可以开始执行 app 代码。
* Chrome site isolation：Chrome 的每个标签页都是一个独立的进程，这其中也有共享初始化资源的技术，这让 chrome 的速度变得很快。
* ……

### Parallel Universe: Backup and Fault-tolerence

有了平行宇宙，我们就有了犯错的底气：我们可以时不时地对状态机 fork，如果某个时刻程序 crash 了，我们就恢复最近一次保存的状态机副本；如果到了下一个存档点仍然没有 crash，我们在 fork 出新的存档的同时可以把上一份存档销毁。

平行宇宙还可以为我们提供容错机制：比如有一些并发 bug 触发的概率很小。那么如果某一次不幸地触发了并发 bug，我们可以利用存档点回到过去，然后再跑一遍，说不定就绕过了 bug 可以继续执行了。

## A `fork()` in the Road

将状态机完整地复制一遍在早期是一件轻量级的事情，但现在随着系统中的机制越来越丰富，fork() 要做的事情越来越繁重：信号、线程、ptrace 等等。fork() 的语义也变得越来越复杂。比如：

* 有了信号机制后，当操作系统给进程发送信号时，子进程是否也接收到这个信号？答案是子进程也会接收到信号，这就设计了 process group 等一系列概念。
* 当线程加入后，fork 是把当前进程的所有线程复制，还是只复制执行 fork 的线程？答案是后者。
* ……

POSIX 给出的一个更安全的创建子进程的 API 是 posix_spawn()：

```c
int posix_spawn(pid_t *pid, const char *path,
                       const posix_spawn_file_actions_t *file_actions,
                       const posix_spawnattr_t *attrp,
                       char *const argv[], char *const envp[]);
```

它有相当复杂的参数。这是一个非常明显的 fork 后时代设计的 API，它打包执行 fork 和 execve，将执行过程分为 fork, pre-exec 和 exec 三个阶段，并在 pre-exec 阶段对 signal handler, file action 等东西做一系列设置。

*A fork() in the Road* 一文中提出了 fork() 的七宗罪：

- Fork is no longer simple - 要考虑的机制越来越多；
- Fork doesn’t compose - 比如 `fork-printf.c`，将标准库中 buffer 的内容复制很可能不是程序员的初衷；
- Fork isn’t thread-safe
- Fork is insecure - fork() 出的子进程和父进程地址空间完全相同，打破了 ASLR；
- Fork is slow 
- Fork doesn’t scale
- Fork encourages memory overcommit

