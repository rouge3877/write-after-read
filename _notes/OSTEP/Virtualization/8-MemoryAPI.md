---
date: 2024-10-31
title: MemoryAPI

---


# 8-Memory API

## **THE CRUX: HOW TO ALLOCATE AND MANAGE MEMORY**

> In UNIX/C programs, understanding how to allocate and manage
> memory is critical in building robust and reliable software. What inter-
> faces are commonly used? What mistakes should be avoided?

## 1.`melloc` and `free`

They are just C library calls...

## 2. `brk` and `sbrk`

>   One such system call is called `brk`, which is used to change the location of the program’s **break**: **<u>the location of the end of the heap</u>**. It takes one argument (the address of the new break), and thus either increases or decreases the size of the heap based on whether the new break is larger or smaller than the current break. An additional call `sbrk` is passed an increment but otherwise serves a similar purpose.
>
>   Note that you should never directly call either `brk` or `sbrk`. They are used by the memory-allocation library; if you try to use them, you will likely make something go (horribly) wrong. Stick to `malloc()` and `free()` instead.

## 3. `mmap()`

>   Finally, you can also obtain memory from the operating system via the `mmap()` call. By passing in the correct arguments, `mmap()` can create an **anonymous memory region** within your program — a region which is not associated with any particular file but rather with **swap space**, something we’ll discuss in detail later on in virtual memory. This memory can then also be treated like a **heap** and managed as such. Read the manual page of `mmap()` for more details.

