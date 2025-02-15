---
date: 2024-10-27
title: Scheduling MLFQ

---

# 5-Scheduling: Multi-Level Feedback Queue

## **Crux: HOW TO SCHEDULE WITHOUT PERFECT KNOWLEDGE?**

|    data    | field | category |
| :--------: | :---: | :------: |
| 2024-10-27 |  OS   |  Notes   |

>   How can we design a scheduler that both minimizes response time for interactive jobs while also minimizing turnaround time without a *priori* knowledge of job length?
>
>   The fundamental problem MLFQ tries to address is two-fold:
>
>   -   It would like to optimize **turnaround time**, which, as we saw in the previous note, is done by running shorter jobs first
>       -   *Unfortunately, the OS doesn’t generally know how long a job will run for, exactly the knowledge that algorithms like SJF (or STCF) require*
>   -   MLFQ would like to make a system feel responsive to interactive users, and thus minimize **response time**
>       -   *Unfortunately, algorithms like Round Robin reduce response time but are terrible for turnaround time*
>
>   **LEARN FORM HISTORY**: The multi-level feedback queue is an excellent example of a system that learns from the past to predict the future. 

## 1. Basic 2 Rules for MLFQ

 MLFQ has a number of distinct **queues**, each assigned a different **priority level**. At any given time, a job that is ready
to run is on a single queue.

*   **Rule 1**: If Priority(A) > Priority(B), A runs (B doesn’t).
*   **Rule 2**: If Priority(A) = Priority(B), A & B run in RR.

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027160019739.png" alt="image-20241027160019739" style="zoom:45%;" />

Rather than giving a fixed priority to each job, MLFQ *varies* the priority of a job based on its ***observed behavior***.

>   For example, a job repeatedly relinquishes the CPU while waiting for input from the key-board, MLFQ will keep its priority high, as this is how an interactive process might behave. If, instead, a job uses the CPU intensively for long periods of time, MLFQ will reduce its priority. In this way, MLFQ will try to learn about processes as they run, and thus *use the history of the job to predict its future behavior*.

## 2. Attempt #1: How To Change Priority

>   Rather than giving a fixed priority to each job, MLFQ *varies* the priority of a job based on its ***observed behavior***.
>
>   *   **CPU-bound** (CPU-intensive): need a lot of CPU time but where response time isn’t important
>   *   **IO-bound** (IO-intensive): mix of interactive jobs that are short-running (and may frequently relinquish the CPU)
>
>   Job’s **allotment**: The allotment is the amount of time a job can spend at a given priority level before the scheduler reduces its priority. For simplicity, at first, we will assume the allotment is *equal to a single time slice*.

*   **Rule 3**: When a job enters the system, it is placed at the **highest priority** (the topmost queue).
*   **Rule 4a**: If a job **uses up** its allotment while running, its priority is **reduced** (i.e., it moves down one queue).
*   **Rule 4b**: If a job **gives up** the CPU (for example, by performing an I/O operation) before the allotment is up, it **stays at the same priority level** (i.e., its allotment is reset).

>    Because it *doesn’t know* whether a job will be a short job or a long-running job, it first *assumes* it might be a short job, thus
>   giving the job high priority. If it actually is a short job, it will run quickly and complete; if it is not a short job, it will slowly move down the queues, and thus soon prove itself to be a long-running more batch-like process. In this manner, **MLFQ approximates SJF**.

## 3. Attempt #2: The Priority Boost

>   NOW😰😰😰: 
>
>   *   **Game the scheduler** => Before the allotment is used, issue an I/O operation and thus relinquish the CPU; doing so allows you to remain in the same queue, and thus gain a higher percentage of CPU time.
>   *   **STRAVATION** => There are “too many” interactive jobs in the system, they will combine to consume all CPU time, and thus long-running jobs will never receive any CPU time (they starve).
>   *   **Alternative** => A program may **change its behavior** over time; what was CPU-bound may transition to a phase of interactivity.

 ***What could we do in order to guarantee that CPU-bound jobs will make some progress (even if it is not much?).***

*   **Rule 5**: After some time period S, move **all** the jobs in the system to the **topmost** queue.
    
    *   Solve **STRAVATION** & **Alternative** below.
    
    *   What should S be set to? ousterhoutS =>  **voo-doo constants**!!!
    
    *   >   TIP: **AVOID VOO-DOO CONSTANTS (OUSTERHOUT’S LAW)**
        >   Avoiding voo-doo constants is a good idea whenever possible. Unfortunately, as in the example above, it is often difficult. One could try to make the system learn a good value, but that too is not straightforward. The frequent result: a configuration file filled with default parameter values that a seasoned administrator can tweak when something isn’t quite working correctly. As you can imagine, these are often left unmodified, and thus we are left to hope that the defaults work well in the field. This tip brought to you by our old OS professor, John Ousterhout, and hence we call it **Ousterhout’s Law**
    



On the left, there is no priority boost, and thus the long-running job gets starved once the two short jobs arrive; on the right, there is a priority boost every 100 ms (which is likely too small of a value, but used here for the example),

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027163213062.png" alt="image-20241027163213062" style="zoom:50%;" />



## 4. Attempt #3: Better Accounting

*   (Rule 4a & Rule 4b =>) **Rule 4**: Once a job **uses up** its time **allotment at a given level** (regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).



Without any protection from gaming (the scheduler with the old Rules 4a and 4b, on the left), a process can issue an I/O before its allotment ends, thus staying at the same priority level, and dominating CPU time. 

With better accounting in place (the new anti-gaming Rule 4, on the right), regardless of the I/O behavior of the process, it slowly moves down the queues, and thus cannot gain an unfair share of the CPU.

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027164632910.png" alt="image-20241027164632910" style="zoom:50%;" />



## 5. Conclusion: History is its guide

**History is its guide: pay attention to how jobs behave over time and treat them accordingly.**

1.    Rule 1: If Priority(A) > Priority(B), A runs (B doesn’t).
2.   Rule 2: If Priority(A) = Priority(B), A & B run in round-robin fashion using the time slice (quantum length) of the given queue.
3.   Rule 3: When a job enters the system, it is placed at the highest priority (the topmost queue).
4.   Rule 4: Once a job uses up its time allotment at a given level (regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).
5.   Rule 5: After some time period S, move all the jobs in the system to the topmost queue.



## 6. Further More

1.   **FreeBSD** scheduler (version 4.3) uses a formula to calculate the current priority level of a job, basing it on how much CPU the process has used.

2.   **ADVICE**: As the operating system rarely knows what is best for each and every process of the system, it is often useful to provide interfaces to allow users or administrators to provide some hints to the OS. 

     We often call such **hints** advice, as the OS need not necessarily pay attention to it, but rather might take the **advice** into account in order to make a better decision.

3.   **Lower Priority, Longer Quanta**: Most MLFQ variants allow for varying time-slice length across different queues. The **high-priority queues are usually given short time slices**; they are comprised of interactive jobs, after all, and thus quickly alternating between them makes sense (e.g., 10 or fewer milliseconds). The low-priority queues, in contrast, contain long-running jobs that are CPU-bound; hence, longer time slices work well (e.g., 100s of ms). 

     <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241027165200748.png" alt="image-20241027165200748" style="zoom:50%;" />
