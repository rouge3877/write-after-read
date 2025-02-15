---
date: 2024-09-29
title: Schedulingc Introduction

---

# 4-Scheduling: Introduction

## **Crux: How to develop scheduling policy**

|    data    | field | category |
| :--------: | :---: | :------: |
| 2024-09-29 |  OS   |  Notes   |

>   How should we develop a basic framework for thinking about scheduling policies? 
>
>   What are the key assumptions? 
>
>   What metrics are important? 
>
>   What basic approaches have been used in the earliest of computer systems?

## 1. Workload Assumptions

*   Process Running on the system: aka **Workload** - ***The more you know about workload, the more fine-tuned your policy can be.***
*   Make the following assumptions (*although they are mostly unrealistic*) about the processes, sometimes called **jobs**, that are running in the system:
    1.   Each job runs for the same amount of time.
    2.   All jobs arrive at the same time.
    3.   Once started, each job runs to completion.
    4.   All jobs only use the CPU (i.e., they perform no I/O)
    5.   The run-time of each job is known.
*   **The above assumptions seem unrealistic. Because we will *<u>RELAX</u>* them as we go, and eventually develop what we will refer to as a Fully-operational scheduling discipline **

## 2. Scheduling Metrics #1

 Compare different scheduling policies: a **scheduling metric**

*   [PERFORMANCE] - **turnaround time** = the time at which the job completes minus the time at which the job
    arrived in the system
    *   $T_{turnaround} = T_{completion} - T_{arrival}$
    *   Because we have assumed that all jobs arrive at the same time, for now $T_{arrival} = 0$ and hence $T_{turnaround} = T_{completion}$
*   [FAIRNESS] - response time(#4)

## 3. Cases: Only use CPU and Workload in known

We knew **job lengths**, and that jobs **only used the CPU**, and our only metric was **turnaround time**:

>   In fact, for a number of early batch computing systems, these types of scheduling algorithms made some sense. 

### 3.1 First In, First Out (FIFO)

<u>**First Come, First Served**</u>

*   🙂Very Simple Example: 
    *   `A` arrived just a hair before `B` which arrived just a hair before `C`. And each job runs for 10 seconds
    *   The average turnaround time: $T_{turnaround} = (10 +20+30)/3 = 20$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929171416961.png" alt="image-20240929171416961" style="zoom:33%;" />
*   😰Example which ***RELAX*** assumption 1:
    *    `A` arrived just a hair before `B` which arrived just a hair before `C`. And `A` runs for 100 seconds while `B` and `C` run for 10 each
    *   The average turnaround time: $T_{turnaround} = (100 +110+120)/3 = 110$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929171858925.png" alt="image-20240929171858925" style="zoom:33%;" />

### 3.2 Shortest Job First (SJF)

<u>**SJF runs the shortest job first, then the next shortest, and so on.**</u>

*   🙂Example which ***RELAX*** assumption 1:

    *    `A` arrived just a hair before `B` which arrived just a hair before `C`. And `A` runs for 100 seconds while `B` and `C` run for 10 each

    *   The average turnaround time: $T_{turnaround} = (120 +10+20)/3 = 50$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929172222028.png" alt="image-20240929172222028" style="zoom:33%;" />

*   😰Example which ***RELAX*** assumption 1 & 2:

    *   `A` arrives at $t = 0$ and needs to run for 100 seconds, whereas `B` and `C` arrive at $t = 10$ and each need to run for 10 second

    *   The average turnaround time: $T_{turnaround} = [100 + (110-20)+(120-10)]/3 = 103.3$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929172738367.png" alt="image-20240929172738367" style="zoom:33%;" />

### 3.3 Shortest Time-to-Completion First (STCF)

>   To address this concern(Example which ***RELAX*** assumption 1 & 2), we need:
>
>   *   ***RELAX* assumption 3** (that jobs must run to completion)
>   *   **Some Machinery: Preempt** When `B` and `C` arrive, it can **preempt** job `A` and decide to run another job, perhaps continuing `A` later

 **<u>Any time a new job enters the system, the STCF scheduler determines which of the remaining jobs (including the new job) has the least time left, and schedules that one.</u>**

*   🙂Example which ***RELAX*** assumption 1 & 2 (and 3):

    *   `A` arrives at $t = 0$ and needs to run for 100 seconds, whereas `B` and `C` arrive at $t = 10$ and each need to run for 10 second
    *   *But we can preempt a job and decide to run another job*

    *   The average turnaround time: $T_{turnaround} = [(120 - 0) + (20 - 10)+(30-10)]/3 = 50$

    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929173601196.png" alt="image-20240929173601196" style="zoom:33%;" />



## 4. Scheduling Metric #2 (Response Time)

>   Thus, if we knew job lengths, and that jobs only used the CPU, and our only metric was turnaround time, `STCF` would be a great policy. In fact, for a number of early batch computing systems, these types of scheduling algorithms made some sense. However, the introduction of time-shared machines changed all that. <u>Now users would sit at a terminal and demand interactive performance from the system as well</u>. And thus, a new metric was born: **response time**.

>   Indeed, imagine sitting at a terminal, typing, and having to wait 10 seconds to see a response from the system just because some other job got scheduled in front of yours: not too pleasant.

*   **Response Time: The time from when the job arrives in a system to the first time it is scheduled**
    *   $T_{response} = T_{firstrun} - T_{arrival}$
*   😰Above policies are Bad for Response Time
    *   `A`, `B`, and `C` arrive at the same time in the system, and that they each wish to run for 5 seconds
    *   The average response time: $T = (0 + 5 + 10) / 3 = 5$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929174354333.png" alt="image-20240929174354333" style="zoom:33%;" />

## 5.  Build a `Response-Time` friendly Scheduler

### 5.1 Round Robin (Time-Slicing)

 **<u>Instead of running jobs to completion, `RR` runs a job for a Time Slice (or scheduling quantum) and then switches to the next job in the run queue.</u>**

*The length of a time slice must be a **multiple** of the timer-interrupt period.*

*   If `Time Slice` is too large, the **response time** will become  longer.
*   If `Time Slice` is too small, too much overhead will be caused by **too many context switches**.

>    Deciding on the length of the time slice presents a trade-off to a system designer, making it long enough to **amortize** the cost of switching without making it so long that the system is no longer responsive.



*   🙂RR is Good For **Response Time**
    *   `A`, `B`, and `C` arrive at the same time in the system, and that they each wish to run for 5 seconds
    *   ***RR with a time-slice of 1 second would cycle through the jobs quickly***
    *   The average response time: $T = (0 + 1 + 2) / 3 = 5$
    *   <img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20240929174415385.png" alt="image-20240929174415385" style="zoom:33%;" />
*   😰RR is Bad For **Turnaround Time**
    *   More generally, any policy (such as RR) that is fair, i.e., that evenly divides the CPU among active processes on a small time scale, will perform poorly on metrics such as turnaround time. 
    *   **Performance and fairness are often at odds in scheduling.**

### 5.2 Incorporating I/O



## 6. No More Oracle

>   Assumption 5 is the worst assumption we could make. 

In fact, in a general-purpose OS (like the ones we care about), the OS usually knows very little about the length of each job

*   How can we build an approach that behaves like SJF/STCF without such a *priori* knowledge?
*   How can we incorporate some of the ideas we have seen with the RR scheduler so that response time is also quite good?
