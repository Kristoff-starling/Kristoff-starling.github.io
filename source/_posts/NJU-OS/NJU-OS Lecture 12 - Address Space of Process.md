---
layout: post
date: 2022-03-27
title: "NJU-OS Lecture 12: Address Space of Process"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Address Space

在 C 程序中，我们可以用指针访问任何可以访问的东西：比如 main() 函数的首地址，这个地址存储的内容就是 main() 的第一条指令。如果我们随便访问一个奇怪的地址，我们会得到段错误；如果我们往 main() 的首地址处写东西，我们也会得到段错误。

<!-- more -->

命令行工具 pmap 可以帮助我们观察一个进程的地址空间。我们以最小的程序 `minimal.S` 为例，在 GDB 中调试时可以用 `info inferiors` 查看进程号，并用 `pmap pid` 来查看地址空间：

```bash
0000000000400000      4K r---- a.out
0000000000401000      4K r-x-- a.out
00007ffff7ff9000     16K r----   [ anon ]
00007ffff7ffd000      8K r-x--   [ anon ]
00007ffffffde000    132K rw---   [ stack ]
ffffffffff600000      4K --x--   [ anon ]
 total              168K
```

0x401000 之后的内容是代码段，这一段有可执行的权限。此外我们在栈上有环境变量，命令行参数等。

我们可以通过 `strace pmap pid` 来观察 pmap 是如何实现的。可以看到 pmap 使用了 openat() 系统调用来打开 procfs 中的 `/proc/pid/maps` 文件。查看这个文件，我们能看到比 pmap 更丰富的地址空间信息。

### Statically Link

我们尝试查看一个静态链接的二进制文件 (空 main() 函数编译得到) 的地址空间：

```bash
0000000000400000      4K r---- a.out
0000000000401000    560K r-x-- a.out
000000000048d000    160K r---- a.out
00000000004b5000     16K r---- a.out
00000000004b9000     12K rw--- a.out
00000000004bc000    140K rw---   [ anon ]
00007ffff7ff9000     16K r----   [ anon ]
00007ffff7ffd000      8K r-x--   [ anon ]
00007ffffffde000    132K rw---   [ stack ]
ffffffffff600000      4K --x--   [ anon ]
 total             1052K
```

可以看到的地址空间的内容更加丰富。第一个段是 ELF header，后面的可读可执行段是代码段，再后面有只读数据段等等。我们可以将其和 ELF 文件的 program header 进行对比：

```bash
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000528 0x0000000000000528  R      0x1000
  LOAD           0x0000000000001000 0x0000000000401000 0x0000000000401000
                 0x000000000008bf1d 0x000000000008bf1d  R E    0x1000
  LOAD           0x000000000008d000 0x000000000048d000 0x000000000048d000
                 0x0000000000027315 0x0000000000027315  R      0x1000
  LOAD           0x00000000000b4908 0x00000000004b5908 0x00000000004b5908
                 0x00000000000059e8 0x00000000000072b8  RW     0x1000 
```

可以看到地址和权限都能对应上。计算机系统的世界里没有魔法。

### Dynamically Link

动态链接的结果会更复杂一些：

```
555555554000-555555555000 r--p  103:0a  /home/starling/a.out
555555555000-555555556000 r-xp  103:0a  /home/starling/a.out
555555556000-555555557000 r--p  103:0a  /home/starling/a.out
555555557000-555555558000 r--p  103:0a  /home/starling/a.out
555555558000-555555559000 rw-p  103:0a  /home/starling/a.out
7ffff7dbe000-7ffff7dc0000 rw-p  00:00 0  // ???
7ffff7dc0000-7ffff7de6000 r--p  103:09  /usr/lib/x86_64-linux-gnu/libc-2.33.so
7ffff7de6000-7ffff7f51000 r-xp  103:09  /usr/lib/x86_64-linux-gnu/libc-2.33.so
7ffff7f51000-7ffff7f9d000 r--p  103:09  /usr/lib/x86_64-linux-gnu/libc-2.33.so
7ffff7f9d000-7ffff7fa0000 r--p  103:09  /usr/lib/x86_64-linux-gnu/libc-2.33.so
7ffff7fa0000-7ffff7fa3000 rw-p  103:09  /usr/lib/x86_64-linux-gnu/libc-2.33.so
7ffff7fa3000-7ffff7fae000 rw-p  00:00 0  // ???
7ffff7fc3000-7ffff7fc7000 r--p  00:00 0 [vvar]
7ffff7fc7000-7ffff7fc9000 r-xp  00:00 0 [vdso]
7ffff7fc9000-7ffff7fca000 r--p  103:09  /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffff7fca000-7ffff7ff1000 r-xp  103:09  /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffff7ff1000-7ffff7ffb000 r--p  103:09  /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffff7ffb000-7ffff7ffd000 r--p  103:09  /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffff7ffd000-7ffff7fff000 rw-p  103:09  /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffffffde000-7ffffffff000 rw-p  00:00 0 [stack]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0 [vsyscall]
```

我们看到多了很多的段，比如 libc 库的段和加载器的段。可以引起我们关注的一个问题是：有两个没有名字的东西是什么？这两个段可读可写不可执行，是否有可能是 bss 段？我们可以做实验验证这一点：

```c
char ch[1 << 30];
int main () {}
```

再次运行并查看，可以看到没有名字的段变大了很多。再做一次实验：

```c
char ch[1 << 30] = {1};
int main () {}
```

可以看到这次编译变慢了很多，生成的可执行文件非常大。查看地址空间，匿名段上方多了一个很大的段，显然是可读可写数据段，匿名段的大小很小。因此我们可以确定没有名字的段就是 bss 段。

另外一个有意思的问题是：vvar 段和 vdso 段是什么？

我们有时候想要执行系统调用，但又不想陷入操作系统内核 (因为这样比较耗时)，vdso 提供了这样一些不陷入内核就可以完成系统调用功能的函数。一个典型的例子是 `__vdso_gettimeofday()`。在使用时，我们并不需要加上 `__vdso` 的前缀，编译器会自动帮我们链接到 vdso 函数。下面是一个例子：

```c
// vdso.c
#include <sys/time.h>
#include <unistd.h>
#include <stdio.h>
#include <time.h>

double gettime() {
  struct timeval t;
  gettimeofday(&t, NULL); // trapless system call
  return t.tv_sec + t.tv_usec / 1000000.0;
}

int main() {
  printf("Time stamp: %ld\n", time(NULL)); // trapless system call
  double st = gettime();
  sleep(1);
  double ed = gettime();
  printf("Time: %.6lfs\n", ed - st);
}
```

`vdso.c` 中调用了函数 time() 和 gettimeofday()，它们都不会陷入内核。我们可以用 GDB 调试它并查看汇编代码。我们发现 time() 函数的从一个奇怪的地方搬了一些数过来，这些数就构成了时间：

```assembly
0x7ffff7fc7c00 <time>:      lea    -0x4b87(%rip),%rax               # 0x7ffff7fc3080
0x7ffff7fc7c07 <time+7>:    lea    -0x1b8e(%rip),%rdx               # 0x7ffff7fc6080
0x7ffff7fc7c0e <time+14>:   cmpl   $0x7fffffff,-0x4b94(%rip)        # 0x7ffff7fc3084
0x7ffff7fc7c18 <time+24>:   cmove  %rdx,%rax
0x7ffff7fc7c1c <time+28>:   mov    0x20(%rax),%rax
0x7ffff7fc7c20 <time+32>:   test   %rdi,%rdi
0x7ffff7fc7c23 <time+35>:   je     0x7ffff7fc7c28 <time+40>
0x7ffff7fc7c25 <time+37>:   mov    %rax,(%rdi)
0x7ffff7fc7c28 <time+40>:   ret 
```

如果我们在 GDB 中直接用 `x` 指令打印那个地址，会发现我们无权访问 (GDB 不允许我们访问另一个进程的地址空间)，但对照 `/proc/pid/maps` 的结果，我们可以看到 time() 函数在 vdso 段，它所抓取的数据在 vvar 段。

```bash
7ffff7fc3000-7ffff7fc7000 r--p 00000000 00:00 0                          [vvar]
7ffff7fc7000-7ffff7fc9000 r-xp 00000000 00:00 0                          [vdso]
```

该虚拟系统调用实现的原理不太复杂：操作系统在 vvar 段维护一个大致时间的变量，每过一会儿去更新一下，系统中所有的进程都可以使用 vvar 段的数据。

## Managing Address Space

进程的地址空间不是一成不变的，在进程执行过程中可以修改。如果我们用 GDB 调试一个动态链接的程序，用 starti 让其暂停在第一条汇编指令 (此时 PC 在加载器中)，再用 pmap 查看地址空间，可以看到此时只有 .so，还没有 libc 相关的段。但执行到 main() 函数时，地址空间中有了 libc 相关的段。

增加/删除/修改地址空间需要系统调用的支持。Linux 提供了 mmap() 系统调用添加映射，munmap() 系统调用删除映射，mprotect() 函数修改映射的权限。

```c
void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
```

手册中有对 mmap() 的详细描述，比如 addr 本身并不指定 mmap() 映射的地址，只会作为一个映射地址的参考；该函数返回的地址是实际映射的地址。有意思的一点是，mmap() 可以直接将文件中的一大块区域映射到地址空间中：事实上文件并不会被立刻搬入内存，而是在使用时才会将要使用的部分搬进来 (缺页异常)。(注：如果不想映射到文件，而只是开辟一段映射空间，可以在 flags 字段使用 MAP_ANONYMOUS，并将 fd 设置为 -1，操作系统会自动将这段填充为 0。)

有了 mmap() 系统调用以后，操作系统的加载器变得非常好写：我们只需要根据 ELF 文件中所写的需要加载的内容一一调用 mmap() 即可。我们可以用 strace 观测一个程序加载、运行时的所有 mmap() 操作。

有了文件映射之后，随之而来的是一系列一致性问题：进程对文件的修改是否需要立即生效？不同进程映射到同一个文件应当共享内存还是各自有本地副本？操作系统还提供了 msync() 系统调用处理同步相关的问题。这才是系统真正的复杂性。

下面是两个 mmap() 的例子：

```c
// mmap-alloc.c
#include <unistd.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>

#define GiB * (1024LL * 1024 * 1024)

int main() {
  volatile uint8_t *p = mmap(NULL, 3 GiB, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
  printf("mmap: %lx\n", (uintptr_t)p);
  if ((intptr_t)p == -1) {
    perror("cannot map");
    exit(1);
  }
  *(int *)(p + 1 GiB) = 114;
  *(int *)(p + 2 GiB) = 514;
  printf("Read get: %d\n", *(int *)(p + 1 GiB));
  printf("Read get: %d\n", *(int *)(p + 2 GiB));
}
```

我们用 mmap() 映射了一点 3G 的内存空间，并在其中某个位置写了东西，并访问这个位置。我们可以看到虽然我们映射了很大的一块空间，但这个系统调用只用了极短的时间 (一些延迟映射相关的技术)。

```python
#!/usr/bin/env python3

# mmap-disk.py

import mmap, hexdump

with open('/dev/sda', 'rb') as fp:
    mm = mmap.mmap(fp.fileno(), prot=mmap.PROT_READ, length=128 << 30)
    hexdump.hexdump(mm[:512])
```

我们可以将整个磁盘映射到地址空间中，并打印第一个扇区的内容，可以看到结尾处标志 MBR 的 0x55aa。

## Isolation of Address Space

虚拟内存机制为我们提供了内存隔离：每个进程只能在自己的地址空间映射中做事，访问别的进程的地址会触发段错误，或者说，在本地的地址空间中根本看不到别的进程的地址。（真的吗？）

### Jinshan Drifter

在访问检查不严格的程序中，其他进程有办法可以侵入，读取和修改数据。这里展示一个类似于“金山游侠”的红警外挂程序：

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <stdint.h>
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdbool.h>

#define LENGTH(arr)  (sizeof(arr) / sizeof(arr[0]))

int n, fd, pid;
uint64_t found[4096];
bool reset;

void scan(uint16_t val) {
  uintptr_t start, kb;
  char perm[16];
  FILE *fp = popen("pmap -x $(pidof dosbox) | tail -n +3", "r"); assert(fp);

  if (reset) n = 0;
  while (fscanf(fp, "%lx", &start) == 1 && (intptr_t)start > 0) {
    assert(fscanf(fp, "%ld%*ld%*ld%s%*[^\n]s", &kb, perm) >= 1);
    if (perm[1] != 'w') continue;

    uintptr_t size = kb * 1024;
    char *mem = malloc(size); assert(mem);
    assert(lseek(fd, start, SEEK_SET) != (off_t)-1);
    assert(read(fd, mem, size) == size);
    for (int i = 0; i < size; i += 2) {
      uint16_t v = *(uint16_t *)(&mem[i]);
      if (reset) {
        if (val == v && n < LENGTH(found)) found[n++] = start + i;
      } else {
        for (int j = 0; j < n; j++) {
	      if (found[j] == start + i && v != val) found[j] = 0;
	    }
      }
    }
    free(mem);
  }
  pclose(fp);

  int s = 0;
  for (int i = 0; i < n; i++) {
    if (found[i] != 0) s++;
  }
  reset = false;
  printf("There are %d match(es).\n", s);
}

void overwrite(uint16_t val) {
  int s = 0;
  for (int i = 0; i < n; i++)
    if (found[i] != 0) {
      assert(lseek(fd, found[i], SEEK_SET) != (off_t)-1);
      write(fd, &val, 2);
      s++;
    }
  printf("%d value(s) written.\n", s);
}

int main() {
  char buf[32];
  setbuf(stdout, NULL);

  FILE *fp = popen("pidof dosbox", "r");
  assert(fscanf(fp, "%d", &pid) == 1);
  pclose(fp);

  sprintf(buf, "/proc/%d/mem", pid);
  fd = open(buf, O_RDWR); assert(fd > 0);

  for (reset = true; !feof(stdin); ) {
    int val;
    printf("(DOSBox %d) ", pid);
    if (scanf("%s", buf) <= 0) { close(fd); exit(0); }
    switch (buf[0]) {
      case 'q': close(fd); exit(0); break;
      case 's': scanf("%d", &val); scan(val); break;
      case 'w': scanf("%d", &val); overwrite(val); break;
      case 'r': reset = true; printf("Search results reset.\n"); break;
    }
  }
}
```

popen() 函数的声明如下：

```c
FILE *popen(const char *command, const char *type);
```

它的作用是创建一个管道，fork 当前进程，并执行 shell 命令 command。type 字段指示了管道的数据流向，“r" 表示当前进程从管道读取 command 的输出，"w" 表示当前进程通过管道向 command 输出。

main() 函数的主要工作是

* 利用 popen() 执行命令 `pidof dosbox`，获得正在运行的 dosbox 进程的进程号，并用 open() 函数打开该进程下的一个文件 `/proc/pid/mem`，获得一个文件描述符 (open() 中的权限要设置为可读可写，因为我们既要读取游戏信息，又要修改数据)。根据 proc 的手册，我们可以通过这个文件访问该进程的地址空间信息。
* 不断从 stdin (控制台) 读取用户输入，对于 `s val` 型输入，在内存中尝试匹配所有的 val。对于 `w val` 型输入，将之前匹配到的所有位置改写为 val。对于 `r` ，重置之前的匹配结果，对于 `q`，退出外挂程序。

scan() 函数负责在 dosbox 的地址空间中匹配所有的 val，其具体工作是：

* 利用 popen() 执行命令 `pmap $(pidof dosbox) | tail -n +3`，`tail -n +3` 可以指定从 pmap 输出结果的第 3 行开始读取。pmap 输出的第一行通常是进程号，第二行是 ELF 文件头，因此从第 3 行开始处理。
* while () 循环访问 dosbox 地址空间的所有可读可写段 (游戏中值得关注的数据，例如金钱、资源，一定是可以修改的)，访问这些段中的每个地址，看其是否等于 val 并修改 found[] 数组。found[] 数组记录了上一次 reset 以来满足所有匹配结果的地址。

overwrite() 函数负责修改数据。它只要将 found[] 中所有地址处的数据改成我们想要的 val 即可。

这个外挂需要动态使用：外挂刚开始也不知道存储金钱的地址在哪里，但我们可以不断花钱，并拿新的钱的剩余量去 scan，做若干轮之后我们就可以锁定金钱的地址。此时再修改该地址的值，便可以拥有钞能力。

### Pseudo-hardware

有一些类似 "2秒17枪" 的外挂可以以人类无法达到的速度大量重复执行任务。这类外挂可以通过写一个驱动模仿假的硬件，或者利用操作系统/窗口管理器的 API 实现。

### Variable Speed Gear

变速齿轮是另外一类常见的外挂，可以调节游戏的进度 (比如加快动画播放的速度，减小延迟等)。其实现的机制通常和代码注入有关：我们需要找到类似于 gettimeofday() 等时间相关 API 的函数位置，然后改写这部分代码，让它跳转到我们自己编写的函数，这样就可以任意调节时间了。

### Hooking

我们不仅可以修改数据，还可以修改代码，这就是代码注入技术 (hooking)。下面是一个简单的给程序打热补丁 (dynamic software update) 的示例：

```c
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <stdint.h>
#include <assert.h>
#include <stdbool.h>
#include <unistd.h>

void foo()     { printf("In old function %s\n", __func__); }
void foo_new() { printf("In new function %s\n", __func__); }

struct jmp {
  uint32_t opcode: 8;
  int32_t  offset: 32;
} __attribute__((packed));

#define JMP(off) ((struct jmp) { 0xe9, off - sizeof(struct jmp) })
#define PG_SIZE sysconf(_SC_PAGESIZE)

static inline bool within_page(void *addr) {
  return (uintptr_t)addr % PG_SIZE + sizeof(struct jmp) <= PG_SIZE;
}

void DSU(void *old, void *new) {
  void *base = (void *)((uintptr_t)old & ~(PG_SIZE - 1));
  size_t len = PG_SIZE * (within_page(old) ? 1 : 2);
  int flags = PROT_WRITE | PROT_READ | PROT_EXEC;

  printf("Dynamically updating...\n"); fflush(stdout);

  if (mprotect(base, len, flags) == 0) {
    *(struct jmp *)old = JMP((char *)new - (char *)old); // **PATCH**
    mprotect(base, len, flags & ~PROT_WRITE);
  } else {
    perror("DSU fail");
  }
}

int main() {
  foo();
  DSU(foo, foo_new);
  foo();
}
```

运行这个程序，我们可以看到虽然两次调用的都是 foo()，但在执行过 DSU() 函数打补丁之后，第二次执行 foo() 其实跳转到了 foo_new() 执行。该程序的原理是改写 foo() 函数的第一条指令，在 foo 地址处写一个间接跳转指令跳到 foo_new()。为了实现这点，代码中的细节有：

* 使用 mprotect() 系统调用修改 foo() 函数所在区域的权限。代码段本身是不可写的，我们要为其加上修改权限。这里的”区域“有一点微妙：如果我们要插入的跳转指令正好横跨了两个页面，则我们需要为连续两个页面开放修改权限。within_page() 函数判断了这一点。
* `*(struct jmp *)old = JMP((char *)new - (char *)old);` 是打补丁的核心语句。JMP 宏的原理是写上一个间接跳转的操作码，然后计算两个函数地址的 delta。

### Protection

防外挂的主要方法是保证控制流和数据流的完整性。事实上，大部分外挂都像是游戏程序的“调试器”：它们不断监听游戏状态并修改游戏数据/代码。我们可以对独立的进程/驱动做完整性检验，还可以拦截向本进程的 ReadProcessMemory 和 WriteProcessMemory 请求，发现后立刻拒绝执行并封号。

其他的一些解决方法包括：从统计学的角度找出不符合规律的操作；让程序在云/沙盒中运行 (即“计算不再信任操作系统")。