---
layout: post
date: 2022-04-19
title: "NJU-OS Lecture 17: Linking and Loading"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Static Loader

### Loader on OS

可执行文件是一个描述了状态机的初始状态的数据结构。加载器根据可执行文件的描述设置好初始状态机。我们很容易写一个静态加载器：<!-- more -->

```c
// loader-static.c
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <assert.h>
#include <elf.h>
#include <fcntl.h>
#include <sys/mman.h>

#define STK_SZ           (1 << 20)
#define ROUND(x, align)  (void *)(((uintptr_t)x) & ~(align - 1))
#define MOD(x, align)    (((uintptr_t)x) & (align - 1))
#define push(sp, T, ...) ({ *((T*)sp) = (T)__VA_ARGS__; sp = (void *)((uintptr_t)(sp) + sizeof(T)); })

void execve_(const char *file, char *argv[], char *envp[]) {
  // WARNING: This execve_ does not free process resources.
  int fd = open(file, O_RDONLY);
  assert(fd > 0);
  Elf64_Ehdr *h = mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, fd, 0);
  assert(h != (void *)-1);
  assert(h->e_type == ET_EXEC && h->e_machine == EM_X86_64);

  Elf64_Phdr *pht = (Elf64_Phdr *)((char *)h + h->e_phoff);
  for (int i = 0; i < h->e_phnum; i++) {
    Elf64_Phdr *p = &pht[i];
    if (p->p_type == PT_LOAD) {
      int prot = 0;
      if (p->p_flags & PF_R) prot |= PROT_READ;
      if (p->p_flags & PF_W) prot |= PROT_WRITE;
      if (p->p_flags & PF_X) prot |= PROT_EXEC;
      void *ret = mmap(
        ROUND(p->p_vaddr, p->p_align),              // addr, rounded to ALIGN
        p->p_memsz + MOD(p->p_vaddr, p->p_align),   // length
        prot,                                       // protection
        MAP_PRIVATE | MAP_FIXED,                    // flags, private & strict
        fd,                                         // file descriptor
        (uintptr_t)ROUND(p->p_offset, p->p_align)); // offset
      assert(ret != (void *)-1);
      memset((void *)(p->p_vaddr + p->p_filesz), 0, p->p_memsz - p->p_filesz);
    }
  }
  close(fd);

  static char stack[STK_SZ], rnd[16];
  void *sp = ROUND(stack + sizeof(stack) - 4096, 16);
  void *sp_exec = sp;
  int argc = 0;

  // argc
  while (argv[argc]) argc++;
  push(sp, intptr_t, argc);
  // argv[], NULL-terminate
  for (int i = 0; i <= argc; i++)
    push(sp, intptr_t, argv[i]);
  // envp[], NULL-terminate
  for (; *envp; envp++) {
    if (!strchr(*envp, '_')) // remove some verbose ones
      push(sp, intptr_t, *envp);
  }
  // auxv[], AT_NULL-terminate
  push(sp, intptr_t, 0);
  push(sp, Elf64_auxv_t, { .a_type = AT_RANDOM, .a_un.a_val = (uintptr_t)rnd } );
  push(sp, Elf64_auxv_t, { .a_type = AT_NULL } );

  asm volatile(
    "mov $0, %%rdx;" // required by ABI
    "mov %0, %%rsp;"
    "jmp *%1" : : "a"(sp_exec), "b"(h->e_entry));
}

int main(int argc, char *argv[], char *envp[]) {
  if (argc < 2) {
    fprintf(stderr, "Usage: %s file [args...]\n", argv[0]);
    exit(1);
  }
  execve_(argv[1], argv + 1, envp);
}
```

编译好 `loader-static.c` 后，使用命令 `./loader ELF文件名` 可以正确地加载并执行一个静态链接的 ELF 文件。我们可以用 strace 证明我们并没有在 loader 中使用 execve() 系统调用，但我们实现了 execve() 的功能。

我们来仔细阅读这份代码：main() 函数的 execve\_() 实现了 execve() 系统调用的功能。不过这个 execve\_() 是我们自己实现的。execve_() 主要做了以下这些事：

* execve_() 获得的参数，file 是要打开的文件名，argv[] 和 envp[] 是参数列表和环境变量列表。我们打开文件并读出 ELF 文件头，对其进行一系列检查 (比如架构是否正确，类型是否是 EXEC 等)。

* 将要加载的段加载到内存中。我们遍历 ELF 文件的程序头表，将那些标有 `PT_LOAD` 的段加载到指定位置 (即 `p->p_vaddr`)。我们使用 mmap() 来完成“加载”，它的本质是将地址空间中的一段区域映射到文件中。这里有一些比较琐碎的细节，比如在使用了 `MAP_FIXED` 标志后，我们必须保证传给 mmap 的地址是对齐的，因此需要 ROUND()，对齐后我们加载的长度也相应要增加，因此有了第二行的 MOD()。

    > **关于 MAP_FIXED**
    >
    > 通常情况下，传给 mmap() 的第一个参数 addr 内核只是将其当作一个 hint，内核会尽量将要加载的内容放到 addr 附近，但不给出任何保证。但如果使用了 MAP_FIXED 标志，内核会将要加载的内容确定地放到 addr 位置，这种情况下，addr 要保证对齐。
    >
    > MAP_FIXED 标志一定要非常小心地使用，在不同的操作系统，内核版本，libc 版本下进程的地址空间布局可能有很大的差异，MAP_FIXED 会使程序的可移植性下降。

    还有一个小细节是：我们要将 `[p->p_filesz, p->p_memsz)`  .bss 节的部分清零。

* 为程序准备一个运行时栈，这里我们直接开了 1 MiB 的数组作为栈，并保存了 argc 的地址待会传给 stack pointer (我们为各种参数准备了 4KiB 的空间)。接下来我们要将 argv[]，envp[]，aux[] 等放到栈上，这些内容都可以在 System V ABI Figure 3.9 的 Initial Process Stack 中找到。这里示例代码定义了一个非常优雅的宏 push()

    ```c
    #define push(sp, T, ...) ({ *((T*)sp) = (T)__VA_ARGS__; sp = (void *)((uintptr_t)(sp) + sizeof(T)); })
    ```

    它的作用是将当前参数放在 sp 的位置，并把 sp 加上 `sizeof(T)` 移动到下一个空白位置。我们这里做出的一个小修改是不将带下划线的环境变量上栈，这可以帮助我们确信我们的程序对各种内容的掌控。

* 最后是几个內联汇编语句，完成的工作是根据 ABI 规定将 %rdx 设置为 0，将栈指针 %rsp 设置为 argc 的地址，最后根据 ELF 的入口地址将 PC 跳转过去 (此处带 `*` 表示绝对跳转)。

这个 loader 存在一个小问题：loader 本身也是一个 ELF，它运行起来的时候地址空间中本身就有自己的映射。我们进行新状态机的映射时可能会出现映射冲突的情况，如果强行映射可能会使 loader 崩溃。这时我们应该将 loader 的映射挪一个地方，但这会使 loader 的复杂度大幅上升。为了解决这个问题，我们采用的方法是让 loader 动态链接，动态链接和静态链接的地址空间通常差异很大，不会重叠。

> **Loader on OS 与操作系统设计**
>
> 这是一个完全处在用户态的 loader——我们发现操作系统的 execve() 系统调用其实是多余的。我们在用户态通过 open(), mmap() 等系统调用可以实现 execve() 的功能。这不禁让我们思考：操作系统是不是应该把加载的过程从内核移出来，让用户对加载过程有更强的可定制性？……

### Boot Loader

```c
// bootmain.c
#include <stdint.h>
#include <elf.h>
#include <x86/x86.h>

#define SECTSIZE 512
#define ARGSIZE  1024

static inline void wait_disk(void) {
  while ((inb(0x1f7) & 0xc0) != 0x40);
}

static inline void read_disk(void *buf, int sect) {
  wait_disk();
  outb(0x1f2, 1);
  outb(0x1f3, sect);
  outb(0x1f4, sect >> 8);
  outb(0x1f5, sect >> 16);
  outb(0x1f6, (sect >> 24) | 0xE0);
  outb(0x1f7, 0x20);
  wait_disk();
  for (int i = 0; i < SECTSIZE / 4; i ++) {
    ((uint32_t *)buf)[i] = inl(0x1f0);
  }
}

static inline void copy_from_disk(void *buf, int nbytes, int disk_offset) {
  uint32_t cur  = (uint32_t)buf & ~(SECTSIZE - 1);
  uint32_t ed   = (uint32_t)buf + nbytes;
  uint32_t sect = (disk_offset / SECTSIZE) + (ARGSIZE / SECTSIZE) + 1;
  for(; cur < ed; cur += SECTSIZE, sect ++)
    read_disk((void *)cur, sect);
}

static void load_program(uint32_t filesz, uint32_t memsz, uint32_t paddr, uint32_t offset) {
  copy_from_disk((void *)paddr, filesz, offset);
  char *bss = (void *)(paddr + filesz);
  for (uint32_t i = filesz; i != memsz; i++) {
    *bss++ = 0;
  }
}

static void load_elf64(Elf64_Ehdr *elf) {
  Elf64_Phdr *ph = (Elf64_Phdr *)((char *)elf + elf->e_phoff);
  for (int i = 0; i < elf->e_phnum; i++, ph++) {
    load_program(
      (uint32_t)ph->p_filesz,
      (uint32_t)ph->p_memsz,
      (uint32_t)ph->p_paddr,
      (uint32_t)ph->p_offset
    );
  }
}

static void load_elf32(Elf32_Ehdr *elf) {
  Elf32_Phdr *ph = (Elf32_Phdr *)((char *)elf + elf->e_phoff);
  for (int i = 0; i < elf->e_phnum; i++, ph++) {
    load_program(
      (uint32_t)ph->p_filesz,
      (uint32_t)ph->p_memsz,
      (uint32_t)ph->p_paddr,
      (uint32_t)ph->p_offset
    );
  }
}

void load_kernel(void) {
  Elf32_Ehdr *elf32 = (void *)0x8000;
  Elf64_Ehdr *elf64 = (void *)0x8000;
  int is_ap = boot_record()->is_ap;

  if (!is_ap) {
    // load argument (string) to memory
    copy_from_disk((void *)MAINARG_ADDR, 1024, -1024);
    // load elf header to memory
    copy_from_disk(elf32, 4096, 0);
    if (elf32->e_machine == EM_X86_64) {
      load_elf64(elf64);
    } else {
      load_elf32(elf32);
    }
  } else {
    // everything should be loaded
  }

  if (elf32->e_machine == EM_X86_64) {
    ((void(*)())(uint32_t)elf64->e_entry)();
  } else {
    ((void(*)())(uint32_t)elf32->e_entry)();
  }
}
```

`bootmain.c` 是 AbstractMachine 中加载内核的代码。我们已经知道固件中的代码会帮我们把启动磁盘的第一个扇区 (512B) 的 MBR 搬到一个指定的位置并开始执行，上述代码就是主引导扇区的代码。我们的内核镜像第一个扇区是 MBR，第二、三个扇区存储了传给 main() 函数的参数，后面的部分是内核的 ELF，MBR 代码的工作和之前的 loader on OS 一样，将内核 ELF 中该加载的东西放到指定的地方。

该代码和 loader on OS 不一样的地方在于：操作系统中我们有 mmap() 系统调用，可以将一段地址空间直接映射到文件中的内容，但在 boot loader 中我们没有操作系统。幸运的是我们有对于全部硬件资源的掌控：现在还没有虚拟地址，我们可以直接指定物理地址，指哪打哪。`bootmain.c` 中的 read_disk() 函数负责将一个扇区的内容拷贝到 buf 数组中，从硬盘中读取数据的代码非常琐碎，需要参考硬件 I/O 相关的手册。copy_from_disk() 是对 read_disk() 的进一步封装，剩余的 load_program()，load_elf32/64() 的代码和 loader on OS 没有本质区别。

### Linux Kernel Loader

Linux 内核没什么可怕的，只是我们之前写的 loader 的放大版本。

> **学会使用正确的工具**
>
> 使用 vscode 调试代码，可以充分可视化；函数跳转，查找等会非常方便。

## Dynamic Linking

随着库函数越来越大，我们希望项目能够在运行时再链接，这样我们不需要每个文件都链接 libc 库函数，节省内存空间。此外，如果我们的库函数要打一个安全补丁，在没有动态链接的情况下，系统中所有链接库的文件都要重新编译一遍 (非常可怕)，有了动态链接我们就可以免除这个麻烦。

假设我们要实现一个自己的二进制文件格式，支持动态加载，那么我们需要有以下字段：

```c
DL_HEAD

LOAD("libc.dl") // 加载动态库
IMPORT(putchar) // 加载外部符号
EXPORT(hello)   // 为动态库导出本地符号
    
DL_CODE

hello:
	...
    CALL DSYM(putchar) // DSYM(func) 表示 func 是外部的函数

DL_END
```

假设我们有配套的编译器可以生成这种格式的位置无关代码。我们可以实现这种格式上的“全家桶”工具集：`gcc` 对标 `ld` ，`readdl` 对标 `readelf`，`objdump` 对标 `objdump`，`interp` 用于执行：

```c
// main.S
#include "dl.h"

DL_HEAD

LOAD("libc.dl")
LOAD("libhello.dl")
IMPORT(hello)
EXPORT(main)

DL_CODE

main:
  call DSYM(hello)
  call DSYM(hello)
  call DSYM(hello)
  call DSYM(hello)
  movq $0, %rax
  ret

DL_END
```

`main.S` 加载了 libc 库和 libhello 库，引入了 hello 符号，然后在 main() 函数中调用了 4 次 hello()。

```c
// libhello.S
#include "dl.h"

DL_HEAD

LOAD("libc.dl")
IMPORT(putchar)
EXPORT(hello)

DL_CODE

hello:
  lea str(%rip), %rdi
  mov count(%rip), %eax
  push %rbx
  mov %rdi, %rbx
  inc %eax
  mov %eax, count(%rip)
  add $0x30, %eax
  movb %al, 0x6(%rdi)
loop:
  movsbl (%rbx),%edi
  test %dil,%dil
  je out
  call DSYM(putchar)
  inc  %rbx
  jmp loop
out:
  pop %rbx
  ret

str:
  .asciz "Hello X\n"

count:
  .int 0

DL_END
```

`libhello.S` 加载了 libc 库，引入了 putchar 符号，并为动态库提供了 hello() 函数。hello() 函数的功能很简单：每次调用 putchar() 输出字符串 "Hello X\n"，其中 X 会每轮 +1。

```c
// libc.S
#include "dl.h"
#include <sys/syscall.h>

DL_HEAD

EXPORT(putchar)
EXPORT(exit)

DL_CODE

putchar:
  mov %dil, buf(%rip)
  mov $SYS_write, %rax
  mov $1, %rdi
  lea buf(%rip), %rsi
  mov $1, %rdx
  syscall
  ret
buf:
  .byte 0

exit:
  movq $SYS_exit, %rax
  syscall

DL_END
```

`libc.S` 为动态库提供了 putchar() 函数和 exit() 函数，putchar() 函数通过 write 系统调用输出字符，exit() 函数通过 exit 系统调用退出。

 利用全家桶工具 `dlbox.c`，我们可以分别链接三个汇编文件，生成对应的 `.dl` 文件：

```bash
./dlbox gcc main.S
./dlbox gcc libc.S
./dlbox gcc libhello.S
```

我们可以用 `readdl` 工具查看 dl 文件的内容：

```bas
> ./dlbox readdl main.dl
DLIB file main.dl:

    LOAD  libc.dl
    LOAD  libhello.dl
  EXTERN  hello
000000c0  main
```

可以看到 `main.S` 加载了 libc 和 libhello，导入了外部符号 hello，本地有一个符号 main。

我们可以用 `objdump` 工具查看反汇编代码：

```
> ./dlbox objdump libc.dl
Disassembly of binary libc.dl:

0000000000000000 <putchar>:
00000000  40883D1F000000    mov [rel 0x26],dil
00000007  48C7C001000000    mov rax,0x1
0000000E  48C7C701000000    mov rdi,0x1
00000015  488D350A000000    lea rsi,[rel 0x26]
0000001C  48C7C201000000    mov rdx,0x1
00000023  0F05              syscall
00000025  C3                ret
00000026  00                db 0x00

0000000000000027 <exit>:
00000027  48C7C03C000000    mov rax,0x3c
0000002E  0F05              syscall
```

(注：需要在本地安装 nasm 工具集。)

最后，我们不需要将这些 dl 文件汇集在一起链接，就可以直接运行 main.dl：

```c
> ./dlbox interp main.dl
Hello 1
Hello 2
Hello 3
Hello 4
```

我们现在关注这个动态加载器是如何实现的。首先看 `dl.h`。

```c
// dl.h
#define REC_SZ 32
#define DL_MAGIC "\x01\x14\x05\x14"

#ifdef __ASSEMBLER__
  #define DL_HEAD     __hdr: \
                      /* magic */    .ascii DL_MAGIC; \
                      /* file_sz */  .4byte (__end - __hdr); \
                      /* code_off */ .4byte (__code - __hdr)
  #define DL_CODE     .fill REC_SZ - 1, 1, 0; \
                      .align REC_SZ, 0; \
                      __code:
  #define DL_END      __end:

  #define RECORD(sym, off, name) \
    .align REC_SZ, 0; \
    sym .8byte (off); .ascii name

  #define IMPORT(sym) RECORD(sym:,           0, "?" #sym "\0")
  #define EXPORT(sym) RECORD(    , sym - __hdr, "#" #sym "\0")
  #define LOAD(lib)   RECORD(    ,           0, "+" lib  "\0")
  #define DSYM(sym)   *sym(%rip)
#else
  #include <stdint.h>

  struct dl_hdr {
    char magic[4];
    uint32_t file_sz, code_off;
  };

  struct symbol {
    int64_t offset;
    char type, name[REC_SZ - sizeof(int64_t) - 1];
  };
#endif
```

上半部分是给汇编看的，后半部分是给 C 语言看的 (即 `dlbox.h`)。`dl.h` 是对我们的 DIY 二进制文件格式的一个 specification。我们的二进制文件格式如下：

| header | symbol1 | symbol2 | ...  | symboln | 00...0 | code |
| ------ | ------- | ------- | ---- | ------- | ------ | ---- |

其中每个部分都是 32 字节对齐的。header 的结构如 C 语言代码部分所示。header 中有一个 4B 的魔数用于检查文件格式，一个变量 `file_sz` 记录整个文件的大小，一个变量 `code_off` 记录代码段距离文件开头的 offset。通过汇编部分的宏可以看到，在汇编代码中添加了一些符号后这些值都是很容易算出的。

比较关键的宏是 IMPORT()，EXPORT() 和 LOAD()。可以看到它们的本质是在符号表中添加一个表项。符号表的结构如 C 语言代码部分所示，前 8 个字节是这个符号所在位置与文件开头的 offset (如果这是一个外部符号则暂时填 0，加载的时候由动态加载器补全)，type 表示这是一个内部导出的符号，外部导入的符号还是要加载一个动态库。最后的 name[] 数组记录了名字。将这个结构体和宏定义对比起来看也很容易懂，这里值得注意的一个小细节是：`IMPORT(sym)` 中，直接使用 sym 表示的是汇编文件中定义的 sym 符号，在前面加上 `#` 才表示 sym 本身这个字符串。

另外可以看到，DSYM() 宏是一个 PC 相对跳转。`DSYM(sym)` 会被翻译为 `call *sym(%rip)`。该指令的语义是 `PC = mem[next-pc + offset]`。因此在汇编阶段 PC 和本文件符号表中的 sym 的差值 offset 就能确定。在加载时，随着符号表 sym 里面填写的地址被确定，`call` 就能完成正确的跳转。

```c
// dlbox.c
#include <stdio.h>
#include <string.h>
#include <assert.h>
#include <stdint.h>
#include <stdlib.h>
#include <stdbool.h>
#include <unistd.h>
#include <sys/mman.h>
#include <fcntl.h>
#include "dl.h"

#define SIZE 4096
#define LENGTH(arr) (sizeof(arr) / sizeof(arr[0]))

struct dlib {
  struct dl_hdr hdr;
  struct symbol *symtab; // borrowed spaces from header
  const char *path;
};

static struct dlib *dlopen(const char *path);

struct dlib *dlopen_chk(const char *path) {
  struct dlib *lib = dlopen(path);
  if (!lib) {
    fprintf(stderr, "Not a valid dlib file: %s.\n", path);
    exit(1);
  }
  return lib;
}

// Implementation of binutils

void dl_gcc(const char *path) {
  char buf[256], *dot = strrchr(path, '.');
  if (dot) {
    *dot = '\0';
    sprintf(buf, "gcc -m64 -fPIC -c %s.S && "
      "objcopy -S -j .text -O binary %s.o %s.dl", path, path, path);
    system(buf);
  }
}


void dl_readdl(const char *path) {
  struct dlib *h = dlopen_chk(path);
  printf("DLIB file %s:\n\n", h->path);
  for (struct symbol *sym = h->symtab; sym->type; sym++) {
    switch (sym->type) {
      case '+': printf("    LOAD  %s\n", sym->name); break;
      case '?': printf("  EXTERN  %s\n", sym->name); break;
      case '#': printf(   "%08lx  %s\n", sym->offset, sym->name); break;
    }
  }
}

void dl_objdump(const char *path) {
  struct dlib *h = dlopen_chk(path);
  char *hc = (char *)h, cmd[64];
  FILE *fp = NULL;

  printf("Disassembly of binary %s:\n", h->path);

  for (char *code = hc + h->hdr.code_off; code < hc + h->hdr.file_sz; code++) {
    for (struct symbol *sym = h->symtab; sym->type; sym++) {
      if (hc + sym->offset == code) {
        int off = code - hc - h->hdr.code_off;
        if (fp) pclose(fp);
        sprintf(cmd, "ndisasm - -b 64 -o 0x%08x\n", off);
        fp = popen(cmd, "w");
        printf("\n%016x <%s>:\n", off, sym->name);
        fflush(stdout);
      }
    }
    if (fp) fputc(*code, fp);
  }
  if (fp) pclose(fp);
}

// binutils: interpreter
void dl_interp(const char *path) {
  struct dlib *h = dlopen_chk(path);
  int (*entry)() = NULL;
  for (struct symbol *sym = h->symtab; sym->type; sym++)
    if (strcmp(sym->name, "main") == 0)
      entry = (void *)((char *)h + sym->offset);
  if (entry) {
    exit(entry());
  }
}

struct cmd {
  const char *cmd;
  void (*handler)(const char *path);
} commands[] = {
  { "gcc",     dl_gcc },
  { "readdl",  dl_readdl },
  { "objdump", dl_objdump },
  { "interp",  dl_interp },
  { "",        NULL },
};

int main(int argc, char *argv[]) {
  if (argc < 3) {
    fprintf(stderr, "Usage: %s {gcc|readdl|objdump|interp} FILE...\n", argv[0]);
    return 1;
  }

  for (struct cmd *cmd = &commands[0]; cmd->handler; cmd++) {
    for (char **path = &argv[2]; *path && strcmp(argv[1], cmd->cmd) == 0; path++) {
      if (path != argv + 2) printf("\n");
      cmd->handler(*path);
    }
  }
}

// Implementation of dlopen()

static struct symbol *libs[16], syms[128];

static void *dlsym(const char *name);
static void dlexport(const char *name, void *addr);
static void dlload(struct symbol *sym);

static struct dlib *dlopen(const char *path) {
  struct dl_hdr hdr;
  struct dlib *h;

  int fd = open(path, O_RDONLY);
  if (fd < 0) goto bad;
  if (read(fd, &hdr, sizeof(hdr)) < sizeof(hdr)) goto bad;
  if (strncmp(hdr.magic, DL_MAGIC, strlen(DL_MAGIC)) != 0) goto bad;

  h = mmap(NULL, hdr.file_sz, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE, fd, 0);
  if (h == (void *)-1) goto bad;

  h->symtab = (struct symbol *)((char *)h + REC_SZ);
  h->path = path;

  for (struct symbol *sym = h->symtab; sym->type; sym++) {
    switch (sym->type) {
      case '+': dlload(sym); break; // (recursively) load
      case '?': sym->offset = (uintptr_t)dlsym(sym->name); break; // resolve
      case '#': dlexport(sym->name, (char *)h + sym->offset); break; // export
    }
  }

  return h;

bad:
  if (fd > 0) close(fd);
  return NULL;
}

static void *dlsym(const char *name) {
  for (int i = 0; i < LENGTH(syms); i++)
    if (strcmp(syms[i].name, name) == 0)
      return (void *)syms[i].offset;
  assert(0);
}

static void dlexport(const char *name, void *addr) {
  for (int i = 0; i < LENGTH(syms); i++)
    if (!syms[i].name[0]) {
      syms[i].offset = (uintptr_t)addr; // load-time offset
      strcpy(syms[i].name, name);
      return;
    }
  assert(0);
}

static void dlload(struct symbol *sym) {
  for (int i = 0; i < LENGTH(libs); i++) {
    if (libs[i] && strcmp(libs[i]->name, sym->name) == 0) return; // already loaded
    if (!libs[i]) {
      libs[i] = sym;
      dlopen(sym->name); // load recursively
      return;
    }
  }
  assert(0);
}
```

`dlbox.c` 实现了 dl 格式的工具全家桶。它有很多的功能都是借用了 GNU 工具链实现的。这其中最重要的函数是 dlopen()，它可以打开一个 `.dl` 文件，将其中的内容解析出来，并返回一个 dlib 结构体，dlib 结构体的内容和之前绘制的二进制文件结构相同。

dlopen() 做了如下的事情：

* 调用 open() 打开目标文件，从中读取了 `sizeof(dl_hdr)` 的数据，即把文件头读了出来，进行魔数检查，并获得了整个文件的大小。
* 利用 mmap() 系统调用将整个目标文件加载到内存中 (其本质是 copy-on-write 的，因此即使文件很大速度也很快)，将符号表首地址设置为 header 下面一个，设置好 `h->path`。
* 遍历符号表，对于 EXPORT 类型调用 dlexport() 将当前符号加载时在内存中的绝对地址填写到数据结构中；对于 IMPORT 类型调用 dlsym() 在数据结构中查找地址并将本文件符号表中的 offset 填上正确的值；对于 LOAD 型调用 dlload() 加载一个新的文件，dlload() 的本质是递归地调用 dlopen()，不过它会记录当前已经打开过的文件，保证不会重复加载同一个动态库。

在 dlopen() 的基础上， `gcc` `readdl` `objdump` `interp` 四个工具的实现是简单的：

* `gcc` 工具实际上使用的是外部 `gcc` 中的汇编器，我们要求汇编器生成 64 位架构的位置无关代码，然后用 `objcopy` 工具把二进制文件的代码节复制出来，输出到一个 dl 文件中。
* `readdl` 工具首先调用 dlopen() 打开对应的文件，然后遍历整个符号表，根据 type 打印相应的信息。
* `objdump` 工具首先调用 dlopen() 打开对应文件，然后把反汇编的工作交给 ndisasm 工具，`objdump` 主要负责扫描符号表，看当前地址是不是一个新函数，并把函数名打印出来。
* `interp` 工具首先调用 dlopen() 打开对应文件，然后在符号表中寻找是 main 的符号，跳转到 main 的首地址开始执行，并将 main() 的返回值喂给 exit() 退出 dlbox。

我们很快会发现我们设计的二进制文件格式有很多可改进的地方：

* 我们的字符串名时常达不到上限，这使得二进制文件中有大量的 0。我们应该将所有的名字放在一个字符串常量池中，然后其他地方保存指向字符串的指针。

* 我们每个 `.dl` 文件中只有一个代码段，这个代码段是可读可写可执行的。我们希望有更多的代码段，不同的段有不同的权限，这就有了 program header table。

* 在我们自己的二进制文件格式中，来自外部库的函数需要在前面加上 DSYM() 以确保被编译成一个 `call *sym(%rip)` 指令。但实际情况下我们写代码时并不会标注一个函数究竟时来自外部库的函数还是来自外部编译单元的函数——前者必须要到加载的时候才能确定位置，后者在链接的时候就能确定。因此我们不得不将所有的函数都编译成 DSYM() 的形式。这是一个万能的形式，但这种跳转相较于直接相对于 PC 的跳转多了一次访存，效率低。

    我们可以考虑这样一种处理方式：我们准备一个叫 PLT 的东西存储在可执行文件里，向函数 f 的跳转统一编译为静态的跳转到本文件的 f@plt() 函数的简单跳转。这样链接的时候如果发现 f() 其实是来自外部编译单元的函数，就修改一下跳转的目标函数；如果 f() 是外部库的函数，就在 f@plt() 中加一条 `jmp *off(%rip)` 指令。这样我们编译时可以统一使用静态的跳转指令。加载时填写好各个 symbol 的加载时地址后，jmp 指令就可以正常工作了。

    这里存储各个 symbol 地址，在加载时填写的东西就叫 global offset table (GOT)，用于间接跳转的东西就叫 procedure linkage table (PLT)。我们独立发明了 PLT 和 GOT 的概念！

