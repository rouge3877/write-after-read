# 9-Mechanism: [Address Translation](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf)

## **THE CRUX: HOW TO *EFFICIENTLY* AND *FLEXIBLY* VIRTUALIZE MEMORY**

> What we want are:
>
> *   efficient
> *   control at same time
> *   *flexibility*
>
> How can we build an **efficient** virtualization of memory? How do we provide the **flexibility** needed by applications? How do we maintain **control** over which memory locations an application can access, and thus ensure that application memory accesses are properly restricted? How do we do all of this efficiently?

*   Hardware + Operating System
    *    **hardware-based address translation** (ddress translation)
    *   The hardware alone **cannot** virtualize memory, as it just provides the l**ow-level mechanism** for doing so efficiently.
*   **Interposition**: *Interposition is a generic and powerful technique that is  often used to great effect in computer systems*
*   Create a beautiful **illusion**: that the program has its own private memory, where its own code and data reside. 



## 1. First Attempts

### 1.1 Assumptions

*   The user’s address space must be placed contiguously in physical memory. We will also assume, for simplicity, that the size of the address space is not too big; specifically, that **it is less than the size of physical memory**.
*   Each address space is **exactly the same size**.

### 1.2 Dynamic (Hardware-based) Relocation

>    ***Thus, we have the problem: how can we relocate this process in memory in a way that is transparent to the process?*** 

**<u>base and bounds</u>**

*   physical address = virtual address + base
*   use bound to achieve protection

 <u>**Mmemory management unit (MMU)**</u>

*   As we develop more sophisticated memory management techniques, we will be **adding more circuitry** to the MMU.

### 1.3 Hardware and OS Issues

| Hardware Requirements                                        | Notes                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Privileged mode                                              | Needed to prevent user-mode processes from executing privileged operations |
| Base/bounds registers                                        | Need pair of registers per CPU to support address translation and bounds checks |
| Ability to translate virtual addresses and check if within bounds | Circuitry to do translations and check limits; in this case, quite simple |
| Privileged instruction(s) to update base/bounds              | OS must be able to set these values before letting a user program run |
| Privileged instruction(s) to register exception handlers     | OS must be able to tell hardware what code to run if exception occurs |
| Ability to raise exceptions                                  | When processes try to access privileged instructions or out-of-bounds memory |

| OS Requirements        | Notes                                                        |
| ---------------------- | ------------------------------------------------------------ |
| Memory management      | Need to allocate memory for new processes;<br/>Reclaim memory from terminated processes;<br/>Generally manage memory via **free list** |
| Base/bounds management | Must set base/bounds properly upon context switch;<br/>Store on **Process Control Block** |
| Exception handling     | Code to run when exceptions arise;<br/>likely action is to terminate offending process |

### 1.4 Weakness

Because the process stack and heap are not too big, all of the space between the two is simply **wasted**. This type of waste is usually called ***internal fragmentation***, as the space inside the allocated unit is not all used (i.e., is fragmented) and thus wasted.



## 2. Relationship with `Limited Direct Execution`

>   **In most cases, the OS just sets up the hardware appropriately and lets the process run directly on the CPU; only when the process misbehaves does the OS have to become involved.**

*   That means "**<u>*Control at critical time but not all days to keep Efficiency*</u>**"

| OS @ run (kernel mode)                                       | Hardware                                                     | Program (user mode)                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------ |
| **To start process A:**<br/>--allocate entry<br/>----in process table<br/>--`alloc` memory for process<br/>--set base/bound registers<br/>--**return-from-trap** (into A) |                                                              |                                            |
|                                                              | restore registers of A<br/>move to **user mode**<br/>jump to A’s (initial) PC |                                            |
|                                                              |                                                              | **Process A runs**<br/>--Fetch instruction |
|                                                              | translate virtual address<br/>perform fetch                  |                                            |
|                                                              |                                                              | Execute instruction                        |
|                                                              | if explicit load/store:<br/>--ensure address is legal<br/>translate virtual address<br/>perform load/store |                                            |
|                                                              |                                                              | (A runs...)                                |
|                                                              | **Timer interrupt**<br/>move to **kernel mode**<br/>jump to handler |                                            |
| **Handle timer**<br/>decide: stop A, run B<br/>call `switch()` routine<br/>--save regs(A)<br/>----to proc-struct(A)<br/>--(including base/bounds)<br/>--restore regs(B)<br/>----from proc-struct(B)<br/>--(including base/bounds)<br/>**return-from-trap** (into B) |                                                              |                                            |
|                                                              | restore registers of B<br/>move to **user mode**<br/>jump to B’s PC |                                            |
|                                                              |                                                              | Process B runs<br/>--Execute bad load      |
|                                                              | Load is out-of-bounds;<br/>move to **kernel mode**<br/>jump to trap handler |                                            |
| **Handle the trap**<br/>--decide to kill process B<br/>--deallocate B’s memory<br/>--free B’s entry<br/>----in process table |                                                              |                                            |

