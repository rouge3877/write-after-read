---
date: 2024-11-05
title: Free Space Management

---


# 11-[Free Space Management](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)

## THE CRUX: HOW TO MANAGE FREE SPACE

**Free Space Management**: a fundamental aspect of any memory management system:

*   whether it be a `malloc` library (managing pages of a process’s heap) 
*   or the **OS** itself (managing portions of the address space of a process)

**External fragmentation**: the free space gets chopped into little pieces of different sizes and is thus fragmented; subsequent requests may fail because there is no single contiguous space that can satisfy the request, even though the total amount of free space exceeds the size of the request.

>   How should free space be managed, when satisfying variable-sized requests? What strategies can be used to minimize fragmentation? What are the time and space overheads of alternate approaches?

## 1. Assumptions

Most of this discussion will focus on the great history of allocators found in **user-level** memory-allocation libraries:

1.   We assume a basic interface such as that provided by `malloc()` and `free()`:

     *   the user, when **freeing** the space, does not inform the library of its **size**

2.   The generic data structure used to manage free space in the heap is some kind of **free list**. This structure contains references to all of the free chunks of space in the managed region of memory. Of course, this data structure **need not be a list per se**, but just some kind of data structure to track free space.

3.   That primarily we are concerned with **external fragmentation** but not the problem of **internal fragmentation**

4.    Once memory is handed out to a client, it **cannot be relocated to another location** in memory

     *   Thus, **no compaction** of free space is possible

5.   The allocator manages a contiguous region of bytes

     *   >   In some cases, an allocator could ask for that region to grow; for example, a user-level memory-allocation library might call into the kernel to grow the heap (via a system call such as `sbrk`) when it runs out of space. However, ***for simplicity, we’ll just assume that the region is a single fixed size throughout its life.***



## 2. Low-Level Mechanisms

### 2.1 Splitting and Coalescing

*   The allocator will perform an action known as **splitting**: it will find a free chunk of memory that can satisfy the request and split it into two.
*   With **coalescing**, an allocator can better ensure that **large** free extents are available for the application.

### 2.2 Tracking The Size Of Allocated Regions

>   You might have noticed that the interface to `free(void *ptr)` does not **take a size parameter**

Most allocators store a little bit of extra information in a **<u>header</u> block** which is kept in memory, usually just before the handed-out chunk of memory. 

*   The header minimally contains the **size** of the allocated region; it may also contain additional pointers to **speed up deallocation**, **<u>*a magic number to provide additional integrity checking*</u>**, and other information. 

*   Let’s assume a simple header which contains the **size** of the region and a **magic** number, like this:

    ```c
    typedef struct {
        int size;
        int magic;
    } header_t;
    
    void free(void *ptr) {
    	header_t *hptr = (header_t *) ptr - 1;
    	......
        assert(hptr->magic == 1234567)  //as a sanity check 
    }
    ```

*   The size of the free region is the **<u>size of the header *plus* the size of the space allocated</u>** to the user

    >   Thus, when a user requests **N** bytes of memory, the library does not search for a free chunk of size N ; rather, it searches for a free chunk of size **N plus the size of the header**.

### 2.3 Embedding A Free List

**You need to build the list inside the free space itself. ! ! !**

```c
typedef struct __node_t {
    int size;
    struct __node_t *next;
} node_t;
```

**Initializes** the heap and puts the first element of the free list inside that space. We are assuming that the heap is built within some free space acquired via a call to the system call `mmap()`; this is not the only way to build such a heap but serves us well in this example.

```c
// mmap() returns a pointer to a chunk of free space
node_t *head = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
					MAP_ANON|MAP_PRIVATE, -1, 0);
head->size = 4096 - sizeof(node_t);
head->next = NULL;
```

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202170955858.png" alt="image-20241202170955858" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202171012162.png" alt="image-20241202171012162" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202171048615.png" alt="image-20241202171048615" style="zoom:33%;" />



### 2.4 Growing The Heap

Most traditional allocators start with a small-sized heap and then re- quest more memory from the OS when they run out. Typically, this means they make some kind of system call (e.g., `sbrk` in most `UNIX` systems) to grow the heap, and then allocate the new chunks from there. To service the `sbrk` request, the OS finds free physical pages, maps them into the address space of the requesting process, and then returns the value of the end of the new heap; at that point, a larger heap is available, and the request can be successfully serviced.



## 3. Basic Strategies

*We will not describe a “best” approach, but rather talk about some basics and discuss their pros and cons*

1.   **Best Fit**: By returning a block that is close to what the user asks, best fit tries to **reduce wasted space**.

     *   Naive implementations pay a **heavy performance penalty** when performing an exhaustive search for the correct free block

2.   **Worst Fit**: Find the **largest** chunk and return the requested amount; keep the remaining (large) chunk on the free list

     *   Worst fit tries to thus leave big chunks free instead of lots of small chunks that can arise from a best-fit approach
     *   Leading to **excess fragmentation** while still having high overheads
     *   A full search of free space is required, and thus this approach can be **costly**

3.   **First Fit**: Simply finds the first block that is big enough and returns the requested amount to the user.

     *    has the advantage of speed

     *   sometimes pollutes the beginning of the free list with small object

         *   >   Thus, how the allocator manages the free list’s order becomes an issue. One approach is to use **address-based ordering**; by keeping the list ordered by the address of the free space, coalescing becomes easier, and fragmentation tends to be reduced.

4.   **Next Fit**: The next fit algorithm keeps an extra pointer to the location within the list where one was looking **last**.

     *   The idea is to **spread the searches for free space** throughout the list more uniformly, thus avoiding splintering
         of the beginning of the list. 
     *   The **performance** of such an approach is quite **similar to first fit**, as an exhaustive search is once again avoided.



## 4. Advanced Approaches

### 4.1 Segregated Lists

**If a *particular* application has one (or a few) *<u>popular-sized request</u>* that it makes, keep a separate list just to manage objects of that size; all *<u>other requests</u>* are forwarded to a more general memory allocator.**

>   By having a chunk of memory dedicated for one particular size of requests, fragmentation is much less of a concern; moreover, allocation and free requests can be served quite quickly when they are of the right size, as no complicated search of a list is required.

How much memory should one dedicate to the pool of memory that serves specialized requests of a given size, as opposed to the general pool? -----------> **SLAB ALLOCATOR**

*   How the **slab allocator** works in kernel memory management:
    -   **Object Caches**: When the kernel **BOOTs**, it creates **object caches** for *frequently used* kernel objects (like locks and file-system inodes). These caches are **segregated free lists** where objects of the same size are stored, making memory allocation and deallocation faster.
    -   **Memory Requests**: If a cache runs low on objects, it requests **slabs** (chunks of memory) from the general memory allocator. These slabs are multiples of the system's **page size** and hold objects of a specific type.
    -   **Reclaiming Memory**: When the objects in a slab are no longer needed (their reference counts drop to zero), the **general allocator** can reclaim the memory to be reused by the system.
    -   **Pre-initialized Objects**: The slab allocator keeps freed objects in a **pre-initialized state**, avoiding the cost of reinitializing them every time they are reused. This reduces overhead and speeds up memory operations.
*   In essence, the slab allocator efficiently manages memory by using specialized object caches and reducing the cost of initializing and destroying objects.

### 4.2 Binary Buddy Allocator

The **binary buddy allocator** is a memory allocation technique designed to simplify **coalescing**. Here's how it works:

1. **Memory Division**: 
   - The free memory is treated as one large block (size `2^N`).
   - When a memory request is made, the free space is recursively **split in half until a block of the required size is found.**

2. **Allocation**:
   - A block of the appropriate size is allocated. This can lead to **internal fragmentation** since blocks are allocated in fixed, power-of-two sizes.

3. **Coalescing**:
   - When a block is freed, the allocator checks if its **buddy block** (a paired block) is free.
   - If the buddy is free, the two blocks are **coalesced** into a larger block.
   - This coalescing process continues recursively until no more merging is possible.

4. **Efficiency**:
   - The key to the buddy allocator's efficiency is that the **buddy blocks** can be easily identified because their addresses differ by **<u>*only one bit*</u>**, determined by the level in the buddy tree.

5. **Drawbacks**:
   - The system can suffer from **internal fragmentation** due to fixed block sizes.

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202172522556.png" alt="image-20241202172522556" style="zoom:33%;" />
