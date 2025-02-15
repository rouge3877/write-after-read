---
date: 2024-09-21
title: Limited Direct Execution

---

# 3-Mechanism: Limited Direct Execution

## **Crux: HOW TO EFFICIENTLY VIRTUALIZE THE CPU WITH CONTROL**

|    data    | field | category |
| :--------: | :---: | :------: |
| 2024-09-21 |  OS   |  Notes   |

>   *   By time sharing the CPU in this manner, virtualization is achieved, but...
>       *   *performance*?
>       *   **CONTORL ? ? ? **
>
>   *    Both hardware and operating-system support will be required

## 1. Basic Technique: Limited Direct Execution

Before to know about `Limited Direct Execution`, lets have a look about the basic direct execution protocol (without any limits, yet)

| Step |                              OS                              |                        Program                         |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------: |
|  1   | 1. Create entry for process list<br/ >2. Allocate memory for program<br/ >3. Load program into memory<br/ >4. Set up stack with `argc`/`argv`<br/ >5. Clear registers<br/ >6. Execute call `main()`<br/ > |                          ---                           |
|  2   |                             ---                              | 1. Run `main()`<br/ >2. Execute return from main<br/ > |
|  3   | 1. Free memory of process<br/ >2. Remove from process list<br/ > |                          ---                           |

But this approach gives rise to a few problems in our quest to virtualize the CPU:

*   How can the OS **make sure the program** doesn’t do anything that we don’t want it to do, while still running it efficiently
*   When we are running a process, **how does the operating system stop it from running and switch to another process**, thus implementing the time sharing we require to virtualize the CPU?

## 2. Problem #1: Restricted Operations

>   **Main Crux: HOW TO PERFORM RESTRICTED OPERATIONS**
>
>   A process must be able to perform I/O and some other restricted operations, but without giving the process complete control over the system. How can the OS and hardware work together to do so?

1.   Restricted Operations

     *   **USER MODE**
         *   Can't issue I/O requests
         *   Doing so would result in the processor **raising an exception and killed by OS**

     *   **KERNEL MODE**
         *   **Code that runs can do what it likes !**
2.   What should a user process do when it wishes to perform some kind of privileged operation? To enable this, virtually all modern hardware provides the ability for user programs to perform a **system call**

     *   To execute a system call, should execute a special instruction: **`trap`** 
         *   ***<u><mark>XXXXXXX[Q from Rouge: Why trap?]XXXXXXX</mark></u>***
     *   When finished, the OS calls a special `return-from-trap` instruction
     *   [x86] **push** the program counter, flags, and a few other registers onto a **<u>per-process</u>** **kernel stack**
         *   and `return-from-trap` will **pop** these values off the stack and resume execution of the user-mode program
         *   Each process has its **<u>own</u> Kernal Stack !!!**


3.   How does the `trap` know **which code to run** inside the OS?

     *   Kernel set up a **trap table** at **boot time** *[Set trap-table is a privileged operation (of course)]*
         *   Tell the hardware what code to run when certain exceptional events occur.
         *   The OS informs the hardware of the locations of these **trap handlers**, usually with some kind of special instruction.
         *   Once the hardware is informed, it remembers the location of these handlers **until the machine is next reboot**.
     *   **System-call number** is assigned to **each system call**
         *   [User code]: places the desired system-call number:
             *   in a register
             *   a specified location on the stack
         *   [OS]: when handling the system call inside the trap handler:
             *   examines this number
             *   ensures it is valid
             *   executes the corresponding code

| Step @ Run |                    OS (Kernal Mode)                     |                           Hardware                           |                     Program (User Mode)                      |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 1 | Create entry for process list<br/>Allocate memory for program<br/>Load program into memory<br/>Setup user stack with `argv`<br/>Fill kernel stack with `reg/PC`<br/>**return-from-trap** |                             ---                              |                             ---                              |
| 2 |                             ---                              | restore regs from kernel stack<br/>move to user mode<br/>jump to `main` |                             ---                              |
| 3 |                             ---                              |                             ---                              | Run `main()`<br/>...<br/>Call system call<br/>**trap** into OS |
| 4 |                             ---                              | save regs to kernel stack<br/>move to kernel mode<br/>jump to `trap handler` |                             ---                              |
| 5 | Handle trap<br/>Do work of syscall<br/>**return-from-trap**  |                             ---                              |                             ---                              |
| 6 |                             ---                              | restore regs from kernel stack<br/>move to user mode<br/>jump to `PC` after **trap** |                             ---                              |
| 7 |                             ---                              |                             ---                              |    ...<br/>return from `main`<br/>**trap** (via `exit()`)    |
| 8 |     Free memory of process<br/>Remove from process list      |                             ---                              |                             ---                              |

| Step @ boot |     OS (kernel mode)      |                  Hardware                  |
| :---------: | :-----------------------: | :----------------------------------------: |
|      1      | **initialize trap table** |                    ---                     |
|      2      |            ---            | remember address of...<br/>syscall handler |

## 3. Problem #2: Switching Between Processes

>   **Main Crux: HOW TO REGAIN CONTROL OF THE CPU**
>
>   How can the operating system **regain control** of the CPU so that it can switch between processes? To be specific, if a process is running on the CPU, this by definition means the OS is not running. If the OS is not running, how can it do anything at all? (hint: it can’t)

### *3.1 A **Cooperative** Approach: Wait For System Calls (legacy)*

*   Transfer control of the CPU to the OS quite frequently by making **system calls**
*   Applications also transfer control to the OS when they **do something illegal**

### 3.2 A **Non-Cooperative** Approach: The OS Takes Control


>   **THE CRUX: HOW TO GAIN CONTROL WITHOUT COOPERATION**
>   How can the OS gain control of the CPU even if processes are not being cooperative? What can the OS do to ensure a rogue process does not take over the machine?

*   **Timer Interrupt**: timer device can be programmed to raise an interrupt every so many milliseconds

*   When the interrupt is raised, the currently running process is halted, and a pre-configured **interrupt handler** in the OS runs

    *   [@BOOT] OS must inform the hardware of **which code to run** when the timer interrupt occurs

    *   [@BOOT] OS must **start the timer**

        *The timer can also be turned off (also a privileged operation), something we will discuss later when we understand concurrency in more detail*

*   Similar to the system-call trap, OS **saves enough of the state of the program** that was running when the interrupt occurred such that a subsequent return-from-trap instruction will **be able to resume the running program correctly**

### 3.3 Saving and Restoring Context

>   OS executes some low-level assembly code to save the general purpose registers, PC, and the kernel stack pointer of the currently-running process, and then restore said registers, PC, and switch to the kernel stack for the soon-to-be-executing process

*   **SAVE** a few register values for the currently-executing process (**onto its kernel stack**)
*   **RESTORE** a few for the soon-to-be-executing process (**from its kernel stack**)
*   *! By switching stacks, the kernel enters the **call to the switch code in the context of one process** (the one that was interrupted) and **returns in the context of another** (the soon-to-be-executing one)*

| Step | OS @ boot <br/>(kernel mode) |                           Hardware                           |
| :--------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 1 |  **initialize trap table**   |                             ---                              |
| 2 |             ---              | 1. remember addresses of...<br/>2. syscall handler<br/>3. timer handler |
| 3 |  **start interrupt timer**   |                             ---                              |
| 4 |             ---              |            1. start timer<br/>2. interrupt CPU in X ms            |

| Step |                  OS @ run<br/>(kernel mode)                  |                           Hardware                           | Program<br/>(user mode) |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :---------------------: |
| 1 |                             ---                              |                             ---                              |    Process A<br/>...    |
| 2 |                             ---                              | 1. **timer interrupt**<br/>2. `save regs(A) → k-stack(A)`<br/>3. move to kernel mode<br/>4. jump to trap handler |           ---           |
| 3 | 1. Handle the trap<br/>2. Call `switch()` routine `{i,ii,iii}`:<br/> `i. save regs(A) → proc_t(A)`<br/>`ii. restore regs(B) ← proc_t(B)`<br/> `iii. switch to k-stack(B)`<br/>3. **return-from-trap (into B)** |                             ---                              |           ---           |
| 4 |                             ---                              | 1. restore `regs(B) ← k-stack(B)`<br/>2. move to user mode<br/>3. jump to B’s `PC` |           ---           |
| 5 |                             ---                              |                             ---                              |    Process B<br/>...    |

**!!! There are two types of register saves/restores that happen during this protocol:**

1.   **Timer Interrupt occurs**: the user registers of the running process are implicitly saved by the **hardware**, using the **kernel stack of that process**

2.   **OS decides to switch**: the kernel registers are explicitly saved by the **software** (i.e., the OS), but this time into memory in the process structure of the process

     *   >    *This action moves the system from running as if it just trapped into the kernel from A to as if it just trapped into the kernel from B.*

 The `xv6` Context Switch Code:

```assembly
# void swtch(struct context *old, struct context *new);
#
# Save current register context in old
# and then load register context from new.
.globl swtch
swtch:
    # Save old registers
    movl 4(%esp), %eax 	# put old ptr into eax
    popl 0(%eax) 		# save the old IP
    movl %esp, 4(%eax) 	# and stack
    movl %ebx, 8(%eax) 	# and other registers
    movl %ecx, 12(%eax)
    movl %edx, 16(%eax)
    movl %esi, 20(%eax)
    movl %edi, 24(%eax)
    movl %ebp, 28(%eax)
    
    # Load new registers
    movl 4(%esp), %eax 	# put new ptr into eax
    movl 28(%eax), %ebp # restore other registers
    movl 24(%eax), %edi
    movl 20(%eax), %esi
    movl 16(%eax), %edx
    movl 12(%eax), %ecx
    movl 8(%eax), %ebx
    movl 4(%eax), %esp 	# stack is switched here
    pushl 0(%eax) 		# return addr put in place
    ret 				# finally return into new ctxt
```

