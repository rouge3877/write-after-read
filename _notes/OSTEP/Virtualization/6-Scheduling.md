---
date: 2024-10-28
title: Scheduling Lottery

---

# 6-Scheduling: Proportional Share

## **CRUX: HOW TO SHARE THE CPU PROPORTIONALLY?**

>   Proportional-share scheduler (fair-share scheduler) is based around a simple concept: ***instead of optimizing for turnaround or response time, a scheduler might instead try to guarantee that each job obtain a certain percentage of CPU time.***
>
>    Every so often, hold a **lottery** to determine which process should get to **run next**; processes that should run more often should be given more chances to win the lottery.
>
>   How can we design a scheduler to share the CPU in a proportional manner? What are the key mechanisms for doing so? How effective are they?

USE RANDOMNESS:

1.   Random often avoids strange corner-case behaviors that a more traditional algorithm may have trouble handling.
2.   Random also is lightweight, requiring little state to track alternatives.
3.   Random can be quite fast.

## 1. Basic Concept: Tickets Represent Your Share

The percent of tickets that a process has represents its share of the system resource in question. Let’s look at an example. 

Imagine two processes, A and B, and further that A has 75 tickets while B has only 25. Thus, what we would like is for A to receive 75% of the CPU and B the remaining 25%.

>   TIP: USE TICKETS TO REPRESENT SHARES
>   One of the most powerful (and basic) mechanisms in the design of lottery (and stride) scheduling is that of the ticket. The ticket is used to represent a process’s share of the CPU in these examples, but can be applied much more broadly. For example, in more recent work on virtual memory management for hypervisors, Waldspurger shows how tickets can be used to represent a guest operating system’s share of memory. ***Thus, if you are ever in need of a mechanism to represent a proportion of ownership, this concept just might be ... (wait for it) ... the ticket.***

## 2. Ticket Mechanisms

1.   **Ticket Currency**

     User A is running two jobs, A1 and A2, and gives them each 500 tickets (out of 1000 total) in A’s currency. User B is running only 1 job and gives it 10 tickets (out of 10 total). The system converts A1’s and A2’s allocation from 500 each in A’s currency to 50 each in the global currency; similarly, B1’s 10 tickets is converted to 100 tickets. The lottery is then held over the global ticket currency (200 total) to determine which job runs.

     ```c
     User A -> 500 (A’s currency) to A1 -> 50 (global currency)
     	   -> 500 (A’s currency) to A2 -> 50 (global currency)
         
     User B ->  10 (B’s currency) to B1 -> 100 (global currency)
     ```

2.   **Ticket Transfer**

     With transfers, a process can temporarily hand off its tickets to another process.

3.   **Ticket Inflation**

     With inflation, a process can temporarily raise or lower the number of tickets it owns. Of course, in a **competitive** scenario with processes that **do not trust one another**, this makes little sense; one greedy process could give itself a vast number of tickets and take over the machine. Rather, inflation can be applied in an environment where a group of processes trust one another; in such a case, if any one process knows it needs more CPU time, it can boost its ticket value as a way to reflect that need to the system, all without communicating with any other processes.

## 3. Implementation

Keep process in a List: <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027172239735.png" alt="image-20241027172239735" style="zoom:33%;" />

```c
// counter: used to track if we’ve found the winner yet
int counter = 0;

// winner: call some random number generator to
// get a value >= 0 and <= (totaltickets - 1)
int winner = getrandom(0, totaltickets);

// current: use this to walk through the list of jobs
node_t *current = head;
while (current) {
    counter = counter + current->tickets;
    if (counter > winner)
        break; // found the winner
    current = current->next;
}
// ’current’ is the winner: schedule it...
```

## 4. How To Assign Tickets?😰

>    One approach is to assume that the users know best; in such a case, each user is handed some number of tickets, and a user can allocate tickets to any jobs they run as desired. However, this solution is a non-solution: it really doesn’t tell you what to do. ***Thus, given a set of jobs, the “ticket-assignment problem” remains open.***😰
>
>   **However, in a virtualized data center (or cloud), where you might like to assign one-quarter of your CPU cycles to the Windows VM and the rest to your base Linux installation, proportional sharing can be simple and effective.**

And Lottery Scheduler can't dual with IO properly

## 5. Stride Scheduling: why use randomness at all?

Why use randomness at all? So, a new approach: 

*   Each job in the system has a stride, which is **inverse** in proportion to the **number of tickets it has**. 
*   At any given time, pick the process to run that has the **lowest pass value** so far; when you run a process, **increment** its pass counter by its **stride**.

```c
curr = remove_min(queue); 	// pick client with min pass
schedule(curr); 			// run for quantum
curr->pass += curr->stride; // update pass using stride
insert(queue, curr); 		// return curr to queue
```

**HOWEVER😰: lottery scheduling has one nice property that stride scheduling does not: *no global state*. Imagine a new job enters in the middle of our stride scheduling example above; what should its pass value be? Should it be set to 0? If so, it will monopolize the CPU. **

## 6. (The Linux) Completely Fair Scheduler

https://www.usenix.org/system/files/conference/atc18/atc18-bouron.pdf

>   The current Linux approach achieves similar goals in an alternate manner. The scheduler, entitled the **Completely Fair Scheduler** (or CFS), implements fair-share scheduling, but does so in a highly efficient and scalable manner.
>
>   Even after aggressive optimization, scheduling uses about 5% of overall datacenter CPU time [K+15]. Reducing that overhead as much as possible is thus a key goal in modern scheduler architecture.

### 6.1 Basic Operation

**GOAL:** to fairly divide a CPU evenly among all competing processes.

1.   **<u>virtual runtime (`vruntime`)</u>**.
     1.   As each process runs, it **accumulates** `vruntime`. In the most **basic** case, each process’s `vruntime` **increases at the same rate**, in proportion with physical (real) time. 
     2.   When a scheduling decision occurs, CFS will pick the process with **the lowest `vruntime`** to run next.
2.   When to **stop**, run the next ? (**CFS manages this tension through various control parameters.**)
     1.   `sched_latency`:  CFS uses this value to determine **how long** one process should run before considering a switch (effectively determining its time slice but in a dynamic fashion). 
          *   A typical `sched_atency` value is 48 (milliseconds); 
          *   CFS divides this value by **the number (`n`) of processes** running on the CPU to determine the time slice for a process. (48 / 4 = 12ms each process)
     2.   `min_granularity`: Avoid Too many processes
          *   CFS will set the time slice of each process to 6 ms instead
3.   CFS utilizes a **periodic timer interrupt**, which means it can only make decisions at fixed time intervals. 
     1.   This interrupt goes off frequently (e.g., every 1 ms), giving CFS a chance to wake up and determine if the current job has reached the end of its run.
     2.   If a job has a time slice that is not a perfect multiple of the timer interrupt interval, that is OK; CFS tracks vruntime precisely, which means that over the long haul, it will eventually approximate ideal sharing of the CPU.



### 6.2 Weighting (Niceness)

a classic UNIX mechanism known as the **nice** level of a process

*   The nice parameter can be set anywhere from **-20 to +19** for a process, with a default of 0. 
*   **Positive** nice values imply **lower** priority 
*   **Negative** values imply **higher** priority

```c
static const int prio_to_weight[40] = {
    /* -20 */ 88761, 71755, 56483, 46273, 36291,
    /* -15 */ 29154, 23254, 18705, 14949, 11916,
    /* -10 */ 9548,  7620,  6100,  4904,  3906,
    /*  -5 */ 3121,  2501,  1991,  1586,  1277,
    /* 	 0 */ 1024,  820,   655,   526,   423,
    /* 	 5 */ 335,   272,   215,   172,   137,
    /*  10 */ 110,   87,    70,    56,    45,
    /*  15 */ 36,    29,    23,    18,    15,
};
```

Now, each process's slice time is not
$$
time\_slice_k=\frac{sched\_latency}{n}, \forall k
$$
but (**Now, `vruntime` has no relationship with `weight`**)
$$
time\_slice_k=\cfrac{weight_k}{\sum\limits_{i=0}^{n-1}weight_i}\cdot sched\_latency \\
vruntime_i' = vruntime_i + \cfrac{weight_0}{weight_i}\cdot runtimes_i, \quad runtimes_i =time\_slice_i^{[0]}+time\_slice_i^{[1]}+...
$$

### 6.3 Using Red-Black Trees (efficiency)

*   CFS does not keep all processes in this structure; rather, only running (or runnable) processes are kept therein. If a process goes to sleep (say, waiting on an I/O to complete, or for a network packet to arrive), it is removed from the tree and kept track of elsewhere.

*   $O(n)$ operation $\rightarrow$ $O(\log n)$ operation

    <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027182740474.png" alt="image-20241027182740474" style="zoom:33%;" />

### 6.4 Dealing With I/O And Sleeping Processes

*   **BAD**😰: If a continuously running job (A) is interrupted by another job (B) that has been asleep for a long time, B's `vruntime` will be significantly behind A's when it wakes. This could lead B to monopolize the CPU, effectively starving A.
*   **SO**: CFS prevents CPU starvation by resetting the `vruntime` of long-sleeping jobs to the minimum `vruntime` of active jobs when they wake. This avoids one job monopolizing the CPU but may limit fair CPU access for short-sleeping jobs.















