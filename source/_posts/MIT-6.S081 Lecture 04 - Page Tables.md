---
layout: post
date: 2022-01-27
title: "MIT-6.S081 Lecture 04: Page Tables"
categories: ["MIT-6.S081 Introduction to Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Address Spaces

Virtual memory helps realize memory isolation. The basic idea is simple: we want each process to have its own address space. The address spaces are independent.

<!-- more -->

## Paging Hardware

We need a method to maintain the mappings of virtual memory to physical memory. The most common method is to maintain page tables.

We should be aware that page table is a hardware mechanism: all the addresses that the CPU gets are virtual addresses. The memory management unit (MMU) is responsible for translating virtual addresses to physical addresses according to the page table and we use the "pa" to access main memory. The page tables are stored somewhere in the main memory and the satp register in the CPU stores the physical address the current page table.

If we maintain the mappings for each virtual address, we will have $2^{64}$ entries and it's obviously unreasonable. Actually we maintain the mappings for **pages**. The size of each page is 4096 bytes. i.e. The lower 12 bits of va are offset in page. We use 39 bits virtual address - 27 bits for VPN and 12 for offset, and we use 56 bits physical address.

Maintaining $2^{27}$ entries is also a huge task. If all the processes store their page table in the memory, the physical memory will soon be used up. The real RISC-V design of page table has multi-level page tables. The 27 bits are divided into 9+9+9. We use the top 9 bits to index the page directory, and we get the physical address of the lower level page directory. Then we use the second 9 bits to index the next page directory... Each page table/directory is stored in a page. Since each page is 4KB in size and each PTE is 8B in size, there are $512=2^9$ entries in a page, that's where the "9" comes from.

The advantage of the page table tree is that we can leave unused page table entry empty. In that case, we don't need to create the lower-level page tables for that entry, and it greatly saves memory space.

The hierarchical page tables make the translation very expensive. In modern design there's a kind of "cache of PTE entries", called **Translation Look-aside Buffer (TLB)**, that helps accelerate the procedure. It stores the recently used [VA,PA] mappings and before translation you can firstly refer to to TLB to see if the mapping exists. 

In most architectures, the TLB is transparent to the OS. i.e. OS doesn't care about the implementation details of TLB. It just needs to be aware of the existence of TLB and when switching page tables, it needs to flush the TLB. (In RISC-V, that's the instruction `sfence.vma()`)

## Code: xv6 VMcode & Layout

In xv6, physical addresses above 0x80000000 are mapped to the DRAM chips. physical addresses below 0x80000000 are used by I/O devices.

For simplicity, the kernel mappings are identity mappings. i.e. va=pa. Pages for kernel stacks and guard pages are exceptions: guard pages are not mapped so they're put in the high address of virtual address space to save physical memory space. Refer to the "Xv6-book Notes" for details.

Refer to the "Xv6 Source code Manual" for details about how the kernel sets up the kernel address space.