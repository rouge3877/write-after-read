---
date: 2024-11-20
title: Paging Introduction

---

# 12-[Introduction to Paging](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf)

## **THE CRUX: HOW TO VIRTUALIZE MEMORY WITH PAGES**

To chop up space into **fixed-sized pieces**. In virtual memory, we call this idea **paging**

*   Correspondingly, we view physical memory as an array of fixed-sized slots called **page frames**; each of these frames can contain a single virtual-memory page
*   It does not lead to **external fragmentation**, as paging (by design) divides memory into fixed-sized units. 
*   Second, it is quite **flexible**, enabling the sparse use of virtual address spaces.

>   How can we virtualize memory with pages, so as to avoid the problems of segmentation? What are the basic techniques? How do we make those techniques work well, with minimal space and time overheads?

## 1. Overview

<u>**Paging provides *flexibility* and *simplifies* memory management by breaking memory into *fixed-size* pages and mapping them efficiently using page tables.**</u>

1. **Virtual and Physical Memory**: 
   - A simple example uses a 64-byte virtual address space with four 16-byte pages and 128-byte physical memory with eight page frames.
   - Virtual pages are mapped to physical frames, with the OS managing this mapping using a **page table**.
2. **Page Table**:
   - The **page table** stores the mapping of virtual pages to physical frames.
   - Each process has its own page table, with different mappings for its virtual memory.
3. **Address Translation**:
   - A virtual address is split into two parts: the **virtual page number (VPN)** and the **offset**.
   - For a 64-byte address space with 16-byte pages, the VPN is 2 bits, and the offset is 4 bits.
   - To translate a virtual address, the system looks up the VPN in the page table to find the corresponding physical frame, then combines it with the offset to form the physical address.

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202184027307.png" alt="image-20241202184027307" style="zoom:33%;" /><img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202184043864.png" alt="image-20241202184043864" style="zoom:33%;" />



## 2. Where Are Page Tables Stored?

1. **Large Size of Page Tables**:
   
   - For a 32-bit address space with 4KB pages, a **20-bit VPN** and **12-bit offset** are used.
   - This results in **2^20 translations** (about a million), requiring **4MB** of memory for a single page table (4 bytes per entry).
   - For 100 processes, the OS would need **400MB** just for page tables.
   
2. **Challenges and Impact on Modern Systems**:
   
   - Page tables are too large to be stored directly in the **MMU (Memory Management Unit)**.
   - *<u>Instead, they are stored in **physical memory** managed by the OS, though in some systems, page tables may be stored in virtual memory or swapped to disk.</u>*
   
   - While modern machines have gigabytes of memory, using large amounts just for page tables remains inefficient, especially in 64-bit systems where the size becomes even more daunting.

## 3. What’s Actually In The Page Table?

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202184310012.png" alt="image-20241202184310012" style="zoom:50%;" />

1. **Page Table Overview**:
   - The **page table** maps **virtual page numbers (VPN)** to **physical frame numbers (PFN)**.
   - The simplest structure is a **linear page table**, where the OS looks up the **page table entry (PTE)** using the VPN.

2. **Page Table Entry (PTE)**:
   - A **PTE** contains various bits, including:
     - **Valid bit**: Indicates if the translation is valid. It helps in managing **sparse address spaces** by marking unused pages as invalid.
     - **Protection bits**: Control access rights, such as **read**, **write**, or **execute**. Violating these rights triggers a trap to the OS.
     - **Present bit**: Indicates whether the page is in **physical memory** or has been **swapped out** to disk.
     - **Dirty bit**: Marks if the page has been modified since loading into memory.
     - **Accessed bit**: Tracks whether the page has been accessed, aiding in **page replacement** decisions.

3. **Example of a PTE (x86 Architecture)**:
   - The **x86 page table entry** includes:
     - **Present bit (P)**, **Read/Write bit (R/W)**, **User/Supervisor bit (U/S)**, and other caching-related bits.
     - **Accessed (A)** and **Dirty (D)** bits.
     - **Page Frame Number (PFN)**, which points to the physical location.
   - **No Separate Valid Bit**:
       - In **x86 architecture**, the **Present bit (P)** combines the roles of the valid and present bits. If `P=0`, a trap is triggered, and the OS decides if the page should be swapped in or if the access is invalid.
   

## 4. Paging: Also Too Slow

Paging requires us to perform one extra memory reference in order to first fetch the translation from the page table. That is **a lot of work!!!**

```c
// Extract the VPN from the virtual address
VPN = (VirtualAddress & VPN_MASK) >> SHIFT

// Form the address of the page-table entry (PTE)
PTEAddr = PTBR + (VPN * sizeof(PTE))

// Fetch the PTE
PTE = AccessMemory(PTEAddr)

 // Check if process can access the page
 if (PTE.Valid == False)
 	RaiseException(SEGMENTATION_FAULT)
 else if (CanAccess(PTE.ProtectBits) == False)
 	RaiseException(PROTECTION_FAULT)
 else
     // Access OK: form physical address and fetch it
     offset = VirtualAddress & OFFSET_MASK
     PhysAddr = (PTE.PFN << PFN_SHIFT) | offset
     Register = AccessMemory(PhysAddr)
```

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202185036817.png" alt="image-20241202185036817" style="zoom:50%;" />