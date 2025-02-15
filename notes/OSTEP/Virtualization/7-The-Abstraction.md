---
date: 2024-10-29
title: Address Spaces

---


# 7-The Abstraction: Address Spaces

## **THE CRUX: HOW TO VIRTUALIZE MEMORY**
> How can the OS build this abstraction of a private, potentially large address space for multiple running processes (all sharing memory) on top of a single, physical memory?

## 1. Hestory
1. Early Systems

2. Multiprogramming and Time Sharing: **Utilization and efficiency**

    *   OS space + User space -> when switching occurs, move all mem into disk

    * > One way to implement time sharing would be to run one process for a short while, giving it full access to all memory (Figure 13.1), then stop it, save all of its state to some kind of disk (including all of physical memory), load some other process’s state, run it for a while, and thus implement some kind of crude sharing of the machin

    * **That's so slow"**

3. The Address Space
    * requires the OS to create an easy to use abstraction of physical memory
    * We call this abstraction the **address space**, and it is **the running program’s view of memory in the system.**



## 2. Goals
1. **Transparency**

2. **Efficiency** (Both in terms of time and space)

3. **Protection** (Isolation)

    > Isolation is a key principle in building reliable systems. If two entities are properly isolated from one another, this implies that one can fail with- out affecting the other. Operating systems strive to isolate processes from each other and in this way prevent one from harming the other. By using memory isolation, the OS further ensures that running programs cannot affect the operation of the underlying OS. Some modern OS’s take iso- lation even further, by walling off pieces of the OS from other pieces of the OS. Such microkernels [BH70, R+89, S+03] thus may provide greater reliability than typical monolithic kernel designs