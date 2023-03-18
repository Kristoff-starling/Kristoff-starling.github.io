---
title: "MIT-6.S081 Lecture 01: Introduction and Examples"
linktitle: '01: Introduction'
type: docs
draft: false
featured: false

diagram: true
math: true

menu:
    mitos-lectures:
        parent: Contents
        weight: 1
---

## Purposes of O/S

* Abstraction of hardware. Higher level interfaces and abstractions that applications can use can enhance convenience and portability (e.g. processes and file systems)
* Multiplexing. Different applications can execute at the same time.
* Isolation and sharing.
* Security. Restrictions on resources applications have access to.
* (Help applications to get) performance.
* Range of different uses.

## O/S Organization

| User:  VIM, CC, SHELL etc.                                   |
| ------------------------------------------------------------ |
| Kernel: File system etc.<br>Processes, Memory allocations, Access control ... |
| Hardware: CPU, RAM, DISK, NET etc.                           |

## API-Kernel

Applications use API-kernel provided by Kernel to jump into Kernel. For example:

```c
fd = open("out", 1);	// fd is the file descriptor
write(fd, "hello\n", 6);
pid = fork();			// fork() API returns the descriptor of the newly created process
```

> **Do high-level programming language like Python directly use system calls?**
>
> Some high-level programming languages focus on portability - its code should be executable on different operating systems so usually they call library functions instead of directly using system calls to ensure its portability. Of course, theses languages provide methods for programmers to directly use system calls.

> **How is jumping to kernel different from ordinary function calls?**
>
> Kernel codes have direct access to sensitive information such as disks, so the privilege level should be carefully maintained during kernel jumping to ensure the safety of data.

## Why hard/interesting? 

* unforgiving environment: different from applications whose environment is provided by O/S, O/S is built directly on the hardware, which is harder.

    (In this course, QEMU is used to simulate CPU and corresponding hardwares.)

* Tensions: there are some trade-offs. For example

    * we want high efficiency so O/S should be close to the H/W, but to support applications we need a correct high-level abstraction.
    * We want powerful O/S services, but applications want simple interfaces.
    * We want flexible API, but for security reasons we need to have restraints.

* Interactions. For example,

    ```c
    fd = open();
    pid = fork();
    ```

    The semantic of `fork()` is to create a copy of the current process. The file descriptor `fd` we acquire at the first line should be accessible by the child process, so interactions between processes are needed.

## Applications using system calls: Examples

### `read()` `write()`

This is `copy.c`, we execute it on xv6:

```c
// copy.c: copy input to output.

#include "kernel/types.h"
#include "user/user.h"

int main ()
{
    char buf[64];
    
    while (1)
    {
        int n = read(0, buf, sizeof(buf));
        if (n <= 0) break;
        write(1, buf, n);
    }
    
    exit(0);
}
```

`copy.c` 's function is that it prints on the screen whatever you input on the screen. Here 0/1 are the file descriptors of stdin/stdout (pervasive UNIX convention), which, in default, connect to the console input/output. Shell ensures that stdin/stdout have been opened when `copy.c` is being executed.

Here `copy.c` doesn't check the return values of system calls for error (e.g. line 16). We should be more careful when coding.

> **What if we replace `sizeof(buf)` by 65?**
>
> O/S will happily read at most 65 bytes from console input, but it may cause unexpected memory error. It's a bug.

### `open()`

stdin and stdout are automatically opened, but we need a method to open files by ourselves, here's another example program `open.c`:

```c
// open.c: create a file, write to it.

#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int main ()
{
    int fd = open("output.txt", O_WRONLY | O_CREATE);
    write(fd, "ooo\n", 4);
    
    exit(0);
}
```

After running `open`, we use `cat output.txt` to see the contents and we'll get `ooo\n`. Here `fd` is the file descriptor indexing into a table inside the kernel. The kernel maintains a table for every running process and the table tells the kernel which file each file descriptor refers to. (NOTE: same file descriptor may refer to different files in different processes because the "table" is different.)

### `fork()`

This is `fork.c`:

```c
// fork.c: create a new process

#include "kernel/types.h"
#include "user/user.h"

int main ()
{
    int pid;
    pid = fork();
    
    printf("fork() returned %d\n", pid);
    if (pid == 0)
    	printf("child\n");
    else
        printf("parent\n");
  	
    exit(0);
}
```

`fork()` returns in both processes. In the original process, `fork()` returns a value greater than zero representing the pid of the child process. In the new process, `fork()` returns zero. Even if the two processes have the same memory, we can discriminate the parent from the child according to the return value of `fork()`.

The printed message is 

```c
ffoorrkk(()) rreettuurrnende d 0
1c9h
ilpda
rent
```

It seems like messy code, but actually the two processes run at the same time and QEMU simulates a multi-core processor for us, so the two processes alternatively print information on the console.

### `exec()`

When a command is typed into the shell, the shell forks a child process and use `exec()` system call to run the application in the child process. `exec()` loads the instructions in the file you specify over the current process, discards its current memory and starts executing those instructions.

`fork()` and `exec()` are always used together to run an application, here is `forkexec.c`:

```c
// forkexec.c: fork then exec

int main ()
{
    int pid, status;
    pid = fork();
    if (pid == 0)
    {
        char *argv[] = { "echo", "THIS", "IS", "ECHO", 0 };
        	// Note: there should be a null pointer NULL/0 at the end of the array
        exec("echo", argv);
        printf("exec failed!\n");
        exit(1);
    }
    else
    {
        printf("parent waiting\n");
        wait(&status);
        	// the exit status of the child process will be recorded in variable status
        printf("the child exited with status %d\n", status);
    }
    exit(0);
}
```

If `echo` is executed successfully, it will print the arguments and use `exit(0)` to exit. If `echo` returns, which means something goes wrong, the child process will exit by `exit(1)` to tell the parent process about the error.

In lots of cases, `fork()` and `exec()` are used together. It seems that a lot of memory space is wasted because after `fork()` copies the memory of the parent process, `exec()` immediately discards it. But with the aid of virtual memory, we can use the tricky "copy on write" technique to solve the issue.

> **Is there any way for the child process to wait for the parent process?**
>
> No.

> **Will "parent waiting\n" always be printed first?**
>
> Not necessary. The parent process and the child process execute concurrently so there outputs may interweave. However, because it takes a lot of machine instructions to discard the memory, load the memory and start `echo`, "parent waiting\n", in most cases, will be printed first.

> **How should we use `wait()` if there are more than one child processes?**
>
> If there are 2 child processes, the parent process should use 2 `wait()` to wait for both of them to exit.

### IO Redirection

IO redirection can be achieved if we do something sophisticated between `fork()` and `exec()`. Here is `redirect.c`: 

```c
// redirect.c: run a command with output redirected

int main()
{
 	int pid;

	pid = fork();
	if(pid == 0){
		close(1);
		open("output.txt", O_WRONLY|O_CREATE);

        char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };
        exec("echo", argv);
        printf("exec failed!\n");
        exit(1);
  	} else {
    	wait((int *) 0);
  	}

  	exit(0);
}
```

In the child process, we firstly close file descriptor 1 and then open "output.txt", the semantic of `open()` is to allocate the least file descriptor that is not being used to the file. Since 0 is still allocated to the console input, file descriptor 1 is allocated to output.txt and `echo` will output things into file 1, i.e. output.txt. It should be noticed that only the child process's IO is redirected, the parent process, i.e. the shell process, stays the same.

The magical thing is that the application `echo` doesn't need to know about the redirection - the only thing it knows is that it should output to file descriptor 1. This is abstraction.