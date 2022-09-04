---
title: "MIT-6.S081 Lecture 08: Page Faults"
linktitle: '08: 页错误'
type: docs
draft: false
featured: false

diagram: true
math: true

menu:
    mitos-lectures:
        parent: Contents
        weight: 8
---

The benefits of virtual memory, besides offering isolation, include levels of indirection. We have already seen some interesting usage of mappings, such as trampoline page and guard page, but these mappings are relatively static. With the aid of page faults, the mappings become dynamic: when page faults occur, the kernel can change page table mappings. Combination of page table and page fault provides enormous amount of flexibility.

The kernel needs necessary information to handle with page faults:

* The faulting virtual address: RISC-V hardware stores it in the stval register.

* The type of page faults, including instruction page fault, load page fault and store page fault. RISC-V manual specifies each type's scause number:

    | page fault type        | scause number |
    | ---------------------- | ------------- |
    | instruction page fault | 12            |
    | load page fault        | 13            |
    | store page fault       | 15            |

* the va of instruction that causes the exception. The PC value is stored in sepc register and transferred into the trap frame.

## Lazy allocation

Xv6 provides `sbrk()` system call for user program to grow or shrink its heap size. Xv6 uses "eager allocation" for `sbrk()`, i.e. it immediately allocates physical memory for the applications. However, it's difficult for applications to estimate the amount of heap space, so they usually ask for more pages than they need. Using page faults, kernel can respond to the system call in a more intelligent manner.	

In lazy allocation, when `sbrk()` is called, we only increase p->sz by n, not allocating any physical pages. When the user program load/store a virtual address later in the heap, page fault exception is triggered. The kernel checks the faulting va in stval register: if `p->stack < faulting va < p->sz`, we call kalloc() to allocate physical page, update the user page table, and re-execute the instruction.

Here's a simple version of code implementation:

In sys_sbrk(), we don't need to call growproc() to allocate physical pages, the only thing we need to do is to modify p->sz:

```diff
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
+ p->sz = p->sz + n;
- if(growproc(n) < 0)
-   return -1;
  return addr;
}
```

Currently, if we boot xv6 and execute shell instruction `echo hello`, we'll get a kernel panic:

```bash
usertrap(): unexpected scause 0x000000000000000f pid=3
			sepc = 0x00000000000012a4 stval=0x0000000000004008
```

The instruction at 0x12a4 is a store instruction, so the value in scause is 0xf. User program shell has 4 pages and the faulting va stored in stval register is just above the stack and belongs to the heap area.

We need to modify the kernel to let xv6 handle with the page fault instead of directly triggering panic:

```diff
void
usertrap(void)
{
  ...
  if(r_scause() == 8){
    // system call
    ...
  } else if((which_dev = devintr()) != 0){
    // ok
+ } else if (r_scause() = 15){
+ 	uint64 va = r_stval();
+ 	printf("page fault %p\n", va);
+ 	uint64 ka = (uint64) kalloc();
+ 	if(ka == 0){
+ 	  p->killed = 1;
+ 	} else{
+ 	  memset((void *)ka, 0, PGSIZE);
+ 	  va = PGROUNDDOWN(va);
+ 	  if (mappages(p->pagetable, va, PGSIZE, ka, PTE_W|PTE_U|PTE_R) != 0)
+ 	  {
+ 	  	kfree((void *)ka);
+ 	  	p->killed = 1;
+ 	  }
+ 	}
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

in usertrap(), we insert a branch for handling with page faults. We read stval register to get the faulting virtual address, allocate a new physical page, and call mappages() to insert mappings in the user page table.

However, we cannot let xv6 run correctly by only those modification. There's another kernel panic message:

```bash
panic: uvmunmap: not mapped
```

uvmunmap() doesn't allow us to unmap pages that hasn't benn allocated, but in lazy allocation this is valid. Therefore we need to modify uvmunmap():

```diff
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
-     panic("uvmunmap: not mapped");
+     continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

## Zero fill in demand

Ususally there are lots of zero pages in the user space. For example, in the ELF file there is a .bss segment, in xv6, when we use `exec()` system call to load a program, the kernel allocate pages for the .bss segment and set the values as zero. 

A clever implementation is: during `exec()`, we only allocate one zero page in the physical memory and map all the zero pages in the virtual memory space to it. Those mappings should be read only. In the future when one zero page needs to be modified, a page fault exception will be triggered, and the kernel should allocate another zero page in the physical memory and modify page table's PTE to map the virtual page being written to the new page (read-and-write).

This "lazy zero page allocation" has at least 2 advantages:

* In some cases, only a small part of the .bss segment will be modified and zero-fill-in-demand saves lots of physical space.
* In `exec()` we don't need to do lots of allocations, which makes `exec()` faster and provides better interact performance between users and the kernel. (Of course, we somehow "postpone" the penalty: a page fault contains hundreds of load/store instructions, which is very expensive.)

## Copy-on-write (COW) fork

An observation is that: when we use `fork()` system call, xv6 creates physical pages and copies all the memory pages in the parent process into child process. Usually, there's an `exec()` system call afterwards and xv6 frees the pages that have just be created.

Copy-on-write fork somehow optimizes it. During a `fork()` system call, we copy the parent process's page table into the child's page table, i.e. let parent and child map to the same physical pages, instead of making copies of physical pages at once. 

However, we should remember that parent and child's modifications cannot be visible to each other. To achieve the isolation, we set the PTEs as read-only PTEs. In the future, when parent/child process makes a modification, a page fault exception will be triggered, and the kernel is responsible for copying the corresponding physical page, updating page tables, changing the PTEs as read-and-write PTEs, and restarting the faulting instruction.

> **Q: How can the kernel tells the copy-on-write page fault from other invalid page faults?**
>
> Almost all the hardware supporting paging, including RISC-V, can address this issue. In RISC-V's PTE format, there are some bits called RSW bits, which are available for kernel use. Kernel can set copy-on-write bit in the RSW area, and when page faults happen, the kernel checks the copy-on-write bit to decide whether it's a valid page fault.

If copy-on-write fork is implemented, there's another things worth attention: in unmodified xv6, one physical page belongs to more or less one process except the trampoline page, which will never be freed. However, in copy-on-write-fork situation, there will be pages belonging to multiple processes, so when a process is being freed, it cannot directly free all the corresponding physical pages since other processes may be using them. A reasonable approach is that, we maintain a "ref count" for each page recording the number of processes using this page, and we free a page when its ref count equals zero.

## Demand paging

For unmodified xv6, In `exec()` system call the kernel loads all the text segment pages and data segment pages into the memory. Accessing files is quite expensive and sometimes the user program doesn't use all its instructions and data.

If demand paging is implemented, in `exec()`, the kernel doesn't load the pages, but set the PTEs as invalid (PTE_V bit removed). So when the user program executes its first instruction, a page fault exception will be triggered. The text/data segment pages are beforehand binded to corresponding files, and in the page fault handler, the kernel copies pages from the file to the memory, updating page tables and restarting the faulting instruction.

In some situation, when a page fault happens and the kernel needs to load a new page into memory, there's no free pages - the memory is used up. In this case, the kernel should firstly choose one page and evict it into the file/disk, and then use the just freed physical page to load the new page. After that, it can restart the faulting instruction.

There are several strategies for choosing the evicted page. The most commonly used strategy is to evict the least recently used page (LRU strategy). Another useful strategy is that the kernel prefers to choose non-dirty page than dirty page, since evicting non-dirty page only requires disabling the corresponding PTE. In hardware, RISC-V's PTE format includes PTE_A bit representing whether the page has been accessed recently, which is helpful for implementing LRU (of course, the operating system should periodically clear all the PTE_A bits). There's also a PTE_D bit representing whether the page has ever been modified.

## Memory-mapped files

Most modern operating system provides `mmap()` system call, which aims at letting user program use load/store instructions to directly read/write files instead of repeatedly using expensive  `read()` / `write()` system calls. The declaration of `mmap()` is

```c
void *mmap(void *va, size_t length, int prot, int flags, int fd, off_t offset);
```

It will map the specified virtual address to the file descriptor. Some operating systems eagerly do the mappings: it loads pages in the files into the memory. However, by utilizing page fault mechanism, we can firstly only maintain a virtual memory area (vma) recording necessary information, and when a paging fault happens, it loads needed pages into memory, and when the file is closed, the kernel should write dirty pages back to the file/disk.

> **Q: What if multiple processes share one file page?**
>
> We don't know the exact order of read/write operations in the two processes, so it's an undefined behavior, we don't need to do extra maintenance. If we implement a file locking in the file system, we can address the synchronizing issue.