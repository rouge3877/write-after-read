---
date: 2024-09-11
title: Processes

---

# 1-Processes

## **Crux: How to provide the illusion of many CPUs?** 

|    data    | field | category |
| :--: | :--: | :--: |
|2024-09-11 | OS | Notes |

>  *The most fundamental abstractions* - **<u>Process: a running program</u>**
>
>   **How to provide the illusion of many CPUs?** 
>  
>   * 	OS virtualizing the CPU: Run one process, then stop it and run another one
>   * 	Timing Sharing
>
>   **To implement virtualization of the CPU, and to implement it well, the OS will need both some low-level machinery and some high-level intelligence**
> 
>   *  	**Low-level mechanisms: low-level methods or protocols that implement a needed piece of functionality**
>       *  	**context switch**
>       *  	**timing share**
>       
>   *  	**High-level policies: high-level algorithms for making some kind of decision within the OS**
>       * 	**scheduling policy**

## 1. The Abstraction: What's A Process

The **process** is the major OS abstraction of a running program. At any point in time, the process can be described by its state: the contents of memory in its **address space**, the contents of CPU registers (including the **program counter** and **stack pointer**, among others), and information about **I/O** (such as open files which can be read or written).

## 2. Process API

*   Create
*   Destroy
*   Wait
*   Miscellaneous Control
*   Get Status

## 3. Process Creation

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240911173700071.png" alt="image-20240911173700071" style="zoom:33%;" />

1.    **Load** program's code and any static data (e.g., initialized variables) into memory
     *   Programs initially reside on disk (or, in some modern systems, flash-based SSDs) in some kind of executable format
     *   Simple OS: loading process is done eagerly(All at once before running the program)
     *   Modern OS: perform the process lazily(loading pieces of code or data only as they are needed during program execution)
2.   **Allocate** program’s **run-time stack**
     *   OS also likely initialize the stack with arguments
     *   Fill in the parameters to the `main()` function: `argc` and the `argv` array
3.    *May also allocate some memory for the program’s **heap***
4.   Related to **input/output** setup
     *   UNIX: each process by default has three open **file descriptors**
5.    Start the program running at the entry point, namely `main()`
      *   jumping to the `main()` routine

## 4. Process States

>   *[Difference between 'Status' and 'State'](https://stackoverflow.com/a/11229919/23018082):*
>
>   *   *status = how are you? [good/bad]*
>   *   *state = what are you doing? [resting/working]*

Three States: a simplified view

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240911175029604.png" alt="State Transitions" style="zoom:33%;" />

1.   **Running**: A process is running on a processor
2.   **Ready**: A process is ready to run, but OS has chosen not to run it
3.   **Blocked**: A process is not ready to run until some other event takes place
     *   When a process initiates an I/O request to a disk, it becomes blocked and thus some other process can use the processor

| Time | Process_0 | Process_1 |                  Notes                  |
| :--: | :-------: | :-------: | :-------------------------------------: |
|  1   |  Running  |   Ready   |                    -                    |
|  2   |  Running  |   Ready   |         Process_0 initiates I/O         |
|  3   |  Blocked  |  Running  | Process_0 is blocked, So Process_1 runs |
|  4   |  Blocked  |  Running  |                    -                    |
|  5   |  Blocked  |  Running  |                    -                    |
|  6   |   Ready   |  Running  |           Process_0 I/O done            |
|  7   |   Ready   |  Running  |           Process_1 now done            |
|  8   |  Running  |     -     |                    -                    |
|  9   |  Running  |     -     |                    -                    |

## 5. Process Structure

*   **To track the state of each process**
*   Referred as **Process List**. Also referred as PCB(Process Control Block), process descriptor.

``` C
// the registers xv6 will save and restore
// to stop and subsequently restart a process
struct context {
    int eip;
    int esp;
    int ebx;
    int ecx;
    int edx;
    int esi;
    int edi;
    int ebp;
};

// the different states a process can be in
enum proc_state { UNUSED, EMBRYO, SLEEPING,
					RUNNABLE, RUNNING, ZOMBIE };

// the information xv6 tracks about each process
// including its register context and state
struct proc {
    char *mem; 					// Start of process memory
    uint sz; 					// Size of process memory
    char *kstack; 				// Bottom of kernel stack
    							// for this process
    enum proc_state state; 		// Process state
    int pid; 					// Process ID
    struct proc *parent; 		// Parent process
    void *chan; 				// If !zero, sleeping on chan
    int killed; 				// If !zero, has been killed
    struct file *ofile[NOFILE]; // Open files
    struct inode *cwd; 			// Current directory
    struct context context; 	// Switch here to run process
    struct trapframe *tf;		// Trap frame for the
    							// current interrupt
};
```



