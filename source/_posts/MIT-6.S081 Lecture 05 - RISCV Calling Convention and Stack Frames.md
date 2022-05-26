---
layout: post
date: 2022-01-28
title: "MIT-6.S081 Lecture 05: RISC-V Calling Convention and Stack Frames"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## C $\rightarrow$ ASM

A C program

```
int main () {
	...
	exit(0);
}
```

cannot be directly understood by a processor. It should go through a "C -> Asm (`.s` files) -> binary (`.o` files) " process.

<!-- more -->

RISC-V is a reduced instruction set while x86/x86-64, which is common in personal computers, is a complex instruction set. CISC has far more instructions and more complex instruction structure than RISC. CISC has some "big" instruction that does lots of things, while in RISC we use several "small" instructions together to achieve that.

RISC-V has a base integer instruction set, which supports basic functions. A RISC-V processor can also choose to include some extension module to support more complicated functions. For example, if a RISC-V processor includes the "F" standard extension, it can support instructions about single-precision floating-point. Extension modules make RISC-V processors to have backward compatibility. If one day, a new extension is added, and the compiler will get the message from the processor and it can use new instructions to do the compilation.

## RISC-V Registers

<img src="https://kristoff-starling.github.io/files/MIT-6.S081-lecture05.png" alt="registers" style="zoom:33%;" />

> **Why `s1` register is separated from other s-series registers?**
>
> RISC-V has a compressed version of instructions which have only 16 bits. In this version, only 8 registers (x8-x15) are used. (Just a guess)

When function A is calling function B, for register `x`:

* If `x` is a caller-saved register, e.g `ra`, A is responsible for saving its value on the stack and B can overwrite `x` without saving it.
* If `x` is a callee-saved register, A don't need to save its value before calling B and B should save `x`'s value on the stack before using it, and before returning, B is responsible for restore `x`'s value.

## Stack Frames

| return address `*`                                           |
| ------------------------------------------------------------ |
| previous fp                                                  |
| saved registers                                              |
| local variables                                              |
| ...                                                          |
| **return address**                                      `fp` register |
| previous fp (pointed to `*`)                                 |
| saved registers                                              |
| local variables                                              |
| ...                                                               `sp` register |

Stack frames may have different sizes and contents, but the general structure is similar: every stack frame begins with the return value of this function and the value of `fp` register in the previous function. `sp` register always points to the bottom of the current stack frame while `fp` register always points to the top of the current stack frame. When `ret` instruction is executed, pseudo code below will be implemented to adjust stack pointers and PC:

```pseudocode
pc = return address
sp = fp + ENTRY_SIZE
fp = previous fp
```

To comply with the calling conventions, the assembly of a function usually consists of a "prologue", a "body part" and a "epilogue". In the prologue, the function modifies `sp` `fp` registers and creates the stack frame (save values of needed registers). In the epilogue, the function modifies `sp` `fp` registers and restore values of callee-saved registers.

For example, the assembly of a simple function `sum_then_double` which calls function `sum_to` is shown below (note: the `.global` sign means that the function can be fetched at any place): 

```assembly
.global sum_then_double
sum_then_double:
	addi sp, sp, -16		# prologue
	sd ra, 0(sp)			
	
	call sum_to
	li t0, 2
	mul a0, a0, t0
	
	ld ra, 0(sp)			# epilogue
	addi sp, sp, 16
	
	ret
```

`ra` is a caller-saved register, so before calling `sum_to`, `sum_then_double` should firstly save the return address on the stack to avoid `ra` being overwritten by `sum_to`. At last, `sum_then_double` restore the value of `ra` from the stack. 

If we delete the prologue and epilogue:

```assembly
.global sum_then_double
sum_then_double:		
	
	call sum_to
	li t0, 2	(*)
	mul a0, a0, t0
	
	ret
```

The processor will fall into a dead loop: `sum_to` modifies `ra`'s value to the address of the next instruction. (*), so when `sum_then_double` execute `ret`, it cannot trace back to its caller and will go to `li t0, 2`.

## GDB Skills

* Use `layout src` `layout asm` `layout reg` to open a tui window and display source code, assembly code or register values. Use `layout split` to display source code and assembly code at the same time. Use `focus src/asm/reg` to switch between different windows.

* Use `i frame` to display information of the current stack frame.

* Use `backtrace/bt` to display previous calling functions. Furthermore, use `i frame #num` to specify a stack frame to print.

* Use `p [variable]` to print values of variables. Notice that modern compilers have compiling optimization and some variables may be optimized out.

    If `x` is a pointer variable, `p *x` can dereference the pointer and print the value.

* Use `i args` to print the arguments of the current function. `p *argv` can print the first argument. If you want to get a complete list of arguments, use `p *argv@argc`.

* Watchpoints are used to monitor variables and gdb stops running as soon as the values of the expressions of the watchpoints change.

* conditional breakpoint is very powerful. For example, if you want to set a breakpoint in a for-loop but only want to trigger it when `i=5`, use `b ... if i==5`.

