---
date: 2024-11-03
title: Segmentation

---

# 10-[Segmentation](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)

## **THE CRUX: HOW TO SUPPORT A LARGE ADDRESS SPACE**

> With the base and bounds registers, the OS can easily relocate processes to different parts of physical memory.
>
> *   **There is a big chunk of “free” space right in the middle, between the stack and the heap.**
>
> How do we support a large address space with (potentially) a lot of free space between the stack and the heap? Note that in our examples, with tiny (pretend) address spaces, the waste doesn’t seem too bad. Imagine, however, a 32-bit address space (4 GB in size); a typical program will only use megabytes of memory, but still would demand that the entire address space be resident in memory.

*   Hardware + Operating System



## 1. Segmentation: Generalized Base/Bounds

**Instead of having just one base and bounds pair in our MMU, why not have a base and bounds <u>*pair per logical segment*</u> of the address space?** (Code Segment, Stack Segment, Heap Segment, .......)

*   With **a base and bounds pair per segment**, we can place each segment independently in physical memory. 
*   Avoid filling physical memory with unused virtual address space:  only **used memory is allocated space** in physical memory, 

### 1.1 Segmentation Fault

**If we tried to refer to an illegal address which is beyond the end of the heap?** 

*   The hardware detects that the address is out of bounds, **traps into the OS**, likely leading to the termination of the offending process. 
*   And now you know the origin of the famous term that all C programmers learn to dread: the **segmentation violation** or **segmentation fault.**

### 1.2 Offset

>   Now let’s look at an address in the heap, virtual address 4200 (again refer to Figure 16.1). If we just add the virtual address 4200 to the base of the heap (34KB), we get a physical address of 39016, which is not the correct physical address. What we need to first do is **extract the offset into the heap**, i.e., which byte(s) in this segment the address refers to. Because the heap starts at virtual address 4KB (4096), the offset of 4200 is actually 4200 minus 4096, or 104. We then take this offset (104) and add it to the base register physical address (34K) to get the desired result: 34920.

 How does it **know the offset into a segment**, and to which segment an address refers?

1.   **Explicit Approach**

     1.   **Chop up** the address space into segments based **on the top few bits** of the virtual address: 

          The hardware simply **takes the first two bits to determine which segment register to use**, and then **takes the next 12 bits as the offset** into the segment.

     2.   >   In our example, then, if the top two bits are 00, the hardware knows the virtual address is in the code segment, and thus uses the code base and bounds pair to relocate the address to the correct physical location. If the top two bits are 01, the hardware knows the address is in the heap, and thus uses the heap base and bounds. Let’s take our example heap virtual address from above (4200) and translate it, just to make sure this is clear. The virtual address 4200, in binary form, can be seen here:
          >
          >   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202145206626.png" alt="image-20241202145206626" style="zoom:33%;" />

     3.   ```c
          // get top 2 bits of 14-bit VA
          Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT
          // now get offset
          Offset = VirtualAddress & OFFSET_MASK
          if (Offset >= Bounds[Segment])
          	RaiseException(PROTECTION_FAULT)
          else
          	PhysAddr = Base[Segment] + Offset
          	Register = AccessMemory(PhysAddr)
          ```

     4.   Some systems put code in the same segment as the heap and thus use only **one bit** to select which segment to use.

     5.   **Issue: *Maximum Size***:

          >   Another issue with using the top so many bits to select a segment is that it limits use of the virtual address space. Specifically, each segment is limited to a maximum size, which in our example is 4KB (using the top two bits to choose segments implies the 16KB address space gets chopped into four pieces, or 4KB in this example). If a running program wishes to grow a segment (say the heap, or the stack) beyond that maximum, the program is out of luck.

2.    **Implicit approach**

     1.   The hardware determines the segment by noticing how the address was formed.
     2.   For example:
          1.    the address was generated from the **program counter** (i.e., it was an instruction fetch), then the address is within the **code segment**
          2.   if the address is based off of the **stack or base pointer**, it must be in the **stack segment**
          3.   any other address must be in the heap.

### 1.3 Stack: a critical difference: it grows backwards (i.e., towards lower addresses)

**Add a bit in hardware for *[<u>Grows Positive?</u>]***

>   In this example, a segment can be 4KB, and thus the correct negative offset is 3KB minus 4KB which equals -1KB. We  simply add the negative offset (-1KB) to the base (28KB) to arrive at the correct physical address: 27K

### 1.4 Support for Sharing

**Add some bits in hardware for *[<u>Protection Bits</u>]***

|  Segment  | Base | Size (max 4K) | Grows Positive? |  Protection  |
| :-------: | :--: | :-----------: | :-------------: | :----------: |
| Code[00]  | 32K  |      2K       |        1        | Read-Execute |
| Heap[01]  | 34K  |      3K       |        1        |  Read-Write  |
| Stack[11] | 28K  |      2K       |        0        |  Read-Write  |

### 1.5 Fine-grained vs. Coarse-grained Segmentation

-   **Coarse-grained segmentation**: Memory is divided into a few large segments (e.g., code, stack, heap). It's simple but less flexible.
-   **Fine-grained segmentation**: Memory is divided into many small segments, offering more flexibility. Systems like **Multics** and the **Burroughs B5000** used thousands of segments, allowing better memory management.
-   **Segment tables** are used to track many smaller segments, enabling the OS to manage memory more efficiently, loading and unloading segments as needed.

The goal of fine-grained segmentation is to improve memory usage and system performance by tracking and managing smaller chunks of memory more precisely.

## 2. OS support

1.    What should the OS do on a **context switch**?

     *   The segment registers must be saved and restored.

2.   What should  OS interaction when segments **grow** (or perhaps shrink)?

     

3.   How OS managing **free space** in physical memory?

     *   >   The general problem that arises is that physical memory quickly becomes full of little holes of free space, making it difficult to allocate new segments, or to grow existing ones. We call this problem **<u>*external fragmentation*</u>** [R69]; see Figure 16.6 (left).

     *   [Solution 1]:  **Compact** physical memory by rearranging the existing segments.

     *   [Solution 2]:  Use a **free-list** management algorithm that tries to keep large extents of memory available for allocation

         *   best-fit
         *   worst-fit
         *   first-fit
         *   buddy algorithm

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202151038210.png" alt="image-20241202151038210" style="zoom:50%;" />
