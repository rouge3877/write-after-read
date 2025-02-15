---
date: 2024-11-15
title: IO Devices
---


# 1-I/O Devices

## **CRUX: HOW TO INTEGRATE I/O INTO SYSTEMS**

>   How should I/O be integrated into systems? What are the general mechanisms? How can we make them efficient?

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202222327963.png" alt="image-20241202222327963" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202222340617.png" alt="image-20241202222340617" style="zoom:33%;" />

## 1. A Canonical Device

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202222423153.png" alt="image-20241202222423153" style="zoom:33%;" />

*   The first is the hardware **interface** it presents to the rest of the system. 
    *   Just like a piece of software, hardware must also present some kind of interface that allows the system software to control its operation. Thus, all devices have some specified interface and protocol for typical interaction
*   The second part of any device is its **internal structure**. 
    *   This part of the device is implementation specific and is responsible for **implementing** the abstraction the device presents to the system

## 2. Protocol

<u>***By reading and writing these registers, the operating system can control device behavior.***</u>

### 2.1 Programmed I/O (PIO and Polling)

```c
While (STATUS == BUSY)
	; // wait until device is not busy
Write data to DATA register
Write command to COMMAND register
	(starts the device and executes the command)
While (STATUS == BUSY)
	; // wait until device is done with your request
```

OS waits until the device is ready to receive a command by **repeatedly** reading the status register; we call this **polling** the device.

However, there are some **inefficiencies and inconveniences involved**. 

*   The first problem you might notice in the protocol is that polling seems inefficient;
*   It wastes a great deal of CPU time just waiting for the (potentially slow) device to complete its activity, instead of switching to another ready process and thus better utilizing the CPU.

### 2.2 Lowering CPU Overhead With Interrupts

**OS can issue a request, put the calling process to sleep, and context switch to another task.**

*   When the device is finally finished with the operation, it will raise a hardware interrupt, causing the CPU to jump into the OS at a predetermined **interrupt service routine** (ISR) or more simply an **interrupt handler**. 
    *   If a device is **fast**, it may be best to **poll**; 
    *   If it is **slow**, **interrupt**
    *   If the speed of the device is **not known**, or sometimes fast and sometimes slow, it may be best to use a **hybrid** that polls for a little while and then, if the device is not yet finished, uses interrupts. This two-phased approach may achieve the best of both worlds.
    *   **Coalescing**: Multiple interrupts can be coalesced into a single interrupt deliver

### 2.3 More Efficient Data Movement With DMA

*With **PIO**, the CPU spends too much time moving data to and from devices by hand. How can we offload this work and thus allow the CPU to be more effectively utilized?*

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202223257082.png" alt="image-20241202223257082" style="zoom:33%;" />

## 3. Methods Of Device Interaction

*   **explicit** I/O instructions: `in`, `out` in x86
    *   Such instructions are usually **privileged**. The OS controls devices, and the OS thus is the only entity allowed to directly communicate with them.
*   **memory-mapped I/O**: 
    *   memory-mapped approach is nice in that no new instructions are needed to support it

## 4. Fitting Into The OS: The Device Driver (abstraction)

<img src="https://raw.githubusercontent.com/rouge3877/ImageHosting/main/image-20241202223456933.png" alt="image-20241202223456933" style="zoom:50%;" />

*The diagram also shows a **raw** interface to devices, which enables special applications (such as a file-system checker, described later, or a disk defragmentation tool) to directly read and write **blocks** without using the file abstraction.*

**DOWNSIDE:** If there is a device that has many special capabilities, but has to present a generic interface to the rest of the kernel, those special capabilities will go **unused**.

## 5. Advanced

*Because the ideas are relatively obvious — no Einsteinian leap is required to come up with the idea of letting the CPU do something else while a slow I/O is pending — perhaps our focus on “who first?” is misguided. What is certainly clear: as people built these early machines, it became obvious that I/O support was needed. Interrupts, DMA, and related ideas are all direct outcomes of the nature of fast CPUs and slow devices; if you were there at the time, you might have had similar ideas.*



**The xv6 IDE Disk Driver (Simplified): **

```c
static int ide_wait_ready() {
    while (((int r = inb(0x1f7)) & IDE_BSY) || !(r & IDE_DRDY))
        ; // loop until drive isn’t busy
    // return -1 on error, or 0 otherwise
}
static void ide_start_request(struct buf *b) {
    ide_wait_ready();
    outb(0x3f6, 0); // generate interrupt
    outb(0x1f2, 1); // how many sectors?
    outb(0x1f3, b->sector & 0xff); // LBA goes here ...
    outb(0x1f4, (b->sector >> 8) & 0xff); // ... and here
    outb(0x1f5, (b->sector >> 16) & 0xff); // ... and here!
    outb(0x1f6, 0xe0 | ((b->dev&1)<<4) | ((b->sector>>24)&0x0f));
    if(b->flags & B_DIRTY){
        outb(0x1f7, IDE_CMD_WRITE); // this is a WRITE
        outsl(0x1f0, b->data, 512/4); // transfer data too!
    } else {
        outb(0x1f7, IDE_CMD_READ); // this is a READ (no data)
    }
}
void ide_rw(struct buf *b) {
    acquire(&ide_lock);
    for (struct buf **pp = &ide_queue; *pp; pp=&(*pp)->qnext)
        ; // walk queue
    *pp = b; // add request to end
    if (ide_queue == b) // if q is empty
        ide_start_request(b); // send req to disk
    while ((b->flags & (B_VALID|B_DIRTY)) != B_VALID)
        sleep(b, &ide_lock); // wait for completion
    release(&ide_lock);
}
void ide_intr() {
    struct buf *b;
    acquire(&ide_lock);
    if (!(b->flags & B_DIRTY) && ide_wait_ready() >= 0)
        insl(0x1f0, b->data, 512/4); // if READ: get data
    b->flags |= B_VALID;
    b->flags &= ˜B_DIRTY;
    wakeup(b); // wake waiting process
    if ((ide_queue = b->qnext) != 0) // start next request
        ide_start_request(ide_queue); // (if one exists)
    release(&ide_lock);
}
```