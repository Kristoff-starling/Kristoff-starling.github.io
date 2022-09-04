---
title: "MIT-6.S081 Lecture 09: Interrupts"
linktitle: '09: 中断'
type: docs
draft: false
featured: false

diagram: true
math: true

menu:
    mitos-lectures:
        parent: Contents
        weight: 9
---

Type `top` in the command line and you can check the memory usage, processes information etc. in the terminal:

```bash
top - 11:57:33 up 1 day,  1:04,  1 user,  load average: 0.21, 0.25, 0.32
Tasks: 354 total,   1 running, 353 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.9 us,  0.7 sy,  0.0 ni, 98.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  15661.3 total,    586.9 free,   2944.0 used,  12130.4 buff/cache
MiB Swap:  19531.0 total,  19528.5 free,      2.5 used.  11403.6 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  
    840 root      20   0  421340  21428  16712 S   9.0   0.1   3:59.57 Network+ 
   4461 starling   9 -11 2834104  17456  13760 S   4.0   0.1  32:30.95 pulseau+ 
    949 root      20   0   10488   6496   4940 S   1.0   0.0  14:10.30 EasyMon+ 
    962 root      20   0  407004   9836   7688 S   1.0   0.1   8:50.53 ECAgent  
 113206 starling  20   0   36.3g  67604  51524 S   0.7   0.4   0:17.59 code     
   4466 starling  20   0   10520   6480   3740 S   0.3   0.0   0:27.20 dbus-da+ 
   4615 starling  20   0 6096848 442400 163960 S   0.3   2.8  18:26.29 gnome-s+ 
   5241 starling  20   0  561596  35788  26204 S   0.3   0.2   5:20.81 sogoupi+ 
 112573 starling  20   0 4934416 716796 396344 S   0.3   4.5   1:10.73 GeckoMa+ 
 113141 starling  20   0   36.3g 165072  55252 S   0.3   1.0   0:08.28 code     
 113678 starling  20   0   40.6g 274404 121660 S   0.3   1.7   4:42.63 Typora   
 116933 starling  20   0 2571540 173676 127868 S   0.3   1.1   0:24.61 Isolate+ 
 120059 root      20   0       0      0      0 I   0.3   0.0   0:00.03 kworker+ 
 120151 starling  20   0   21704   4412   3460 R   0.3   0.0   0:00.77 top      
      1 root      20   0  165108  11200   7684 S   0.0   0.1   0:04.22 systemd  
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.15 kthreadd 
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
```

We can see that there's little free memory left, most memory is used by the buffer cache. That's a very common case in modern operating systems. Leaving too much memory space idle is wasteful, so operating systems usually make full use of the memory space. In this case, handling page fault exception and evicting pages out to the disk is very frequent.

---

Sometimes the hardware wants attention, so they generate interrupts. The software should save its work, process the interrupts, and afterwards resume its work. The saving and resuming procedures are similar to those of system calls and traps, they all use the same mechanism. But the following features make interrupts different and be worth of a lecture:

* Asynchronous: interrupts can happen at any time. A system call must happen when a user program is running, but when interrupts come, we have no idea which process is being executed, or whether there is a process running.
* Concurrency. We have to address a lot of issues about concurrency and parallelism. This is the start point of talking about concurrency. The devices and the CPU are independent parts: CPU executes instructions while devices handling I/O events. This is true parallelism.
* Device programming: devices like ether-net cards and UART needs programming. There may be manual illustrating the function of each control register, but devices are often pool documented. 

## RISC-V Interrupt Structure

The first question is: where are interrupts come from? Interrupts come from the various devices. Devices send the interrupts to a device called platform-level interrupt controller (PLIC), CPU cores can access PLIC to see whether there's an pending interrupts. When a CPU core finishes processing an interrupt, it sends a signal to the PLIC and PLIC knows that this interrupts can be forgot.

> **Q: Does PLIC has some mechanism to ensure fairness?**
>
> It's up to the kernel. The kernel programs the PLIC hardware. PLIC doesn't really actively choose service of delivering interrupts. The kernel fetches the interrupts and decides where interrupts should be sent. The kernel can also decide the priority of different kinds of interrupts. There is a huge amount of flexibility.

## Organization of Drivers

Drivers are codes used for managing devices. A typical driver structure includes 2 parts:

* Top part: this part handle system calls related to the device such as `read()` and `write()` system calls. It interacts with the user processes, so things like copyin()/copyout() are done in this part.
* Bottom part: this part serves as the interrupt handler of the corresponding device. When the device raises an interrupt, the kernel catches it and calls functions in the bottom part of its driver to handle the interrupt. It may happen that the current process is not the previous pending process, so the interrupt handler cannot count on any information about the process in the CPU core (e.g. page table).

## Programming Devices

In xv6 we use memory mapped I/O. In this case devices show up at particular addresses in the physical memory space, which are determined by the hardware board manufacturers. The operating systems should know these addresses in order to programming devices. OS can use normal load/store instructions to read/write the control registers of the devices.

## Interrupts on Hardware

The PLIC hardware is responsible for receiving interrupts from different devices and routing them to CPU cores. If a CPU core receives an interrupt and the SIE bit of sstatus register is set (i.e. interrupt enabled), the hardware will do the following things:

* Clear the SIE bit in sstatus register to temporarily disable interrupts.
* store PC value into sepc register.
* save the current mode, and then switch to supervisor mode.
* copy the address in stvec register into PC. In xv6, it goes to usertrap()/kerneltrap().

## Case Study: `$`

We want to know how the prompt `$` shows up in the console. In general, the device puts `$` into the UART, and UART generates an interrupt when the char has been sent.

RISC-V hardware has some supports for interrupts:

* SIE register. The supervisor interrupt enable register has a bit for external interrupt, a bit for software interrupt and a bit for timer interrupt. Here we only focus on external interrupts.
* SSTATUS register. This control register contains a bit that can enable/disable interrupts.
* SIP register. The supervisor interrupt pending register, together with SCAUSE register, contain information about what interrupt is coming.

In main(), xv6 does a bunch of preparations which get ready for interrupts. The true time when interrupt is able to come in is when scheduler() calls intr_on():

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    ...
  }
}
```

intr_on() is responsible for writing SSTATUS_SIE bit into SSTATUS register:

```c
static inline void
intr_on()
{
  w_sstatus(r_sstatus() | SSTATUS_SIE);
}
```

Every CPU core runs scheduler() code, and when intr_on() is called, the corresponding core is able to accept interrupts.

The first user process, `init`, opens the console device and assign 0/1/2 to standard input/output/error.

```c
if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
}
dup(0);  // stdout
dup(0);  // stderr
```

Then in user program `/user/sh.c`, getcmd() function uses fprintf() to print a prompt to stderr. Here fprintf() doesn't know what's behind the file descriptor 2, maybe a file, a pipe or a device.

```c
int
getcmd(char *buf, int nbuf)
{
  fprintf(2, "$ ");
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

fprintf() will eventually use `write()` system call to write a character. Refer to the xv6-book notes for the procedure after that.

## Case Study: `ls`

We want to know how the command `ls` is typed into the console. In general, the keyboard connects to the receive line of UART. When a character is typed, UART generates an interrupt to let the kernel fetches that char.

When a interrupt is received by the kernel, usertrap()/kerneltrap() calls devintr() to check whether there's an interrupt and do corresponding operations. devintr() calls plic_claim() to make the CPU core claim the type of device and call uartintr() to handle UART interrupt. In uartintr(), it calls uartgetc() to get the character `l` and `s`, and calls consoleintr() to put them into buffer.

## Interrupts and Concurrency

Concurrency issues in interrupts are difficult to tackle with. Here are some common concurrency problems:

* Devices and CPU cores run in parallel. It's called producer-consumer parallelism and needs to be carefully managed.
* Interrupts can stop the current program. If it stops the user program, the procedure is similar to traps. However, interrupt can also stop the kernel, which means that the instruction sequence in kernel code may not be continuous either now. We have to carefully enable/disable interrupts in kernel code to avoid crashing.
* The top of drivers and the bottom of drivers may run in parallel. For example, when the shell program use `write()` system call to put a dollar prompt on the display, i.e. the top of the console driver is running, another CPU core may be handling interrupts from uart. The top and bottom share resources such as the buffer array. To make sure each core can see the correct, up-to-date content, we need a mechanism to ensure that only one core can do operations to shared resources at a time. Xv6 uses locks to realize that.

Xv6 uses buffer array to address the producer-consumer parallelism. There are a read pointer and a write pointer for each buffer. The buffer is empty when `r==w`, the buffer is full when `w+1 mod BUFFER_SIZE==r`. The producer puts characters at the end of the queue and updates write pointer, while the consumer reads characters at the beginning of the queue and updates read pointer. Buffer array successfully decouple devices and CPU cores, making them able to run separately at high speed.

> **Q: Is there multiple buffer arrays, one for each CPU core?**
>
> The buffer array is an array declared in `/kernel/uart.c` and is set in RAM. There's only one RAM so there's only one buffer array. There are several CPU cores running processes in parallel, and they may access the UART device at the same time, i.e. access the buffer array in parallel. 
>
> The main task for locking is to serialize accesses. Only the core that acquires the lock has access to the array and the read/write pointers. After it releases the lock, other cores can acquire the lock.

## Interrupt Evolution

Interrupts used to be very fast because hardware is very simple in old times. Nowadays, interrupts are slow because devices are more complicated. They have to do lots of preliminary work before generating an interrupt. On the other hand, kernel design becomes more complicated and it takes longer to handle an interrupt.

For some fast devices, e.g. high performing network card, there may be a stream of packets and the period is even shorter than the interrupt handling time. In this case, we have another strategy - polling. The CPU keeps checking a particular control register, once there's data in, the CPU handles it. For slow devices, polling wastes CPU cycles on checking registers, but if devices are fast, polling can save frequent entry/exit cost.

Nowadays, sophisticated devices can dynamically switch between polling and interrupts.

