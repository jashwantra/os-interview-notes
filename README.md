# Operating System Interview Notes

Comprehensive OS notes for C++ developers preparing for technical interviews. These notes are written in an **explanatory, discussion-style tone** — designed to help you explain concepts conversationally in interviews rather than just recite definitions.

## Table of Contents

| Chapter | Topic | What You'll Learn |
|---------|-------|-------------------|
| [01](./01-process-management.md) | Process Management | Process vs Thread, PCB, Process States, Context Switching, IPC mechanisms |
| [02](./02-cpu-scheduling.md) | CPU Scheduling | FCFS, SJF, Round Robin, Priority, MLFQ — when to use which and why |
| [03](./03-threads-concurrency.md) | Threads & Concurrency | User vs Kernel threads, Threading models, Thread pools, C++ threading |
| [04](./04-synchronization.md) | Synchronization | Critical sections, Mutex, Semaphores, Monitors, C++ sync primitives |
| [05](./05-deadlocks.md) | Deadlocks | Four conditions, Prevention strategies, Banker's Algorithm, C++ solutions |
| [06](./06-memory-management.md) | Memory Management | Paging, Virtual Memory, Demand Paging, Page Replacement, Thrashing |
| [07](./07-file-systems.md) | File Systems | Directory structures, Allocation methods, Unix inodes, Journaling |
| [08](./08-storage-disk.md) | Storage & Disk | Disk scheduling algorithms, RAID levels, SSD vs HDD considerations |
| [09](./09-system-calls-kernel.md) | System Calls & Kernel | User/Kernel mode, System call mechanism, Kernel architectures |
| [10](./10-interview-scenarios.md) | Interview Scenarios | Classic questions with discussion-style answers, C++ & OS relationship |

## How to Use These Notes

**For comprehensive preparation**: Read through all chapters sequentially. Each chapter builds on concepts from previous ones and includes practical C++ examples.

**For quick review**: Jump to the Key Takeaways section at the end of each chapter. These summarize the most important points in a few paragraphs.

**For interview practice**: Chapter 10 contains classic interview questions with detailed, conversational answers. Practice explaining these concepts out loud as if you're discussing with an interviewer.

**For C++ developers**: Each chapter connects OS concepts to C++ code. Understanding how `std::thread` maps to kernel threads, or how `new` relates to `brk()`/`mmap()`, helps you write better code and answer interview questions more effectively.

## Key Concepts Quick Reference

### Process Management
A **process** is a program in execution with its own address space. A **thread** is a lightweight execution unit within a process, sharing memory but having its own stack. Use processes for isolation, threads for fast communication.

### CPU Scheduling
The scheduler decides which process runs next. **FCFS** is simple but causes convoy effect. **SJF** is optimal but impractical. **Round Robin** provides fairness. **MLFQ** adapts to process behavior automatically — it's what modern OS use.

### Synchronization
**Race conditions** occur when threads access shared data without synchronization. Use **mutex** for mutual exclusion, **semaphores** for counting resources, **condition variables** for waiting on conditions. In C++, always use RAII (`std::lock_guard`, `std::scoped_lock`).

### Deadlocks
Four conditions must hold: mutual exclusion, hold and wait, no preemption, circular wait. Prevent circular wait through **lock ordering** — always acquire locks in consistent order. Use `std::scoped_lock` for multiple mutexes.

### Memory Management
**Virtual memory** gives each process the illusion of large, contiguous address space. **Paging** divides memory into fixed-size blocks, eliminating external fragmentation. **Demand paging** loads pages only when accessed. **LRU** is the best practical page replacement algorithm.

### File Systems
Files are named collections of data with metadata. **Hierarchical directories** organize files. **Indexed allocation** (Unix inodes) provides good random access. **Journaling** enables crash recovery.

### Storage
**Disk scheduling** minimizes head movement on HDDs — LOOK/C-LOOK are most practical. **RAID** combines disks for performance and/or reliability. **SSDs** have no seek time, so scheduling matters less but TRIM is essential.

### System Calls
**Dual-mode operation** protects the system — user programs can't access hardware directly. **System calls** are the controlled gateway to kernel services. They have overhead (100-1000 cycles), so batching and buffering help.

## Essential Formulas

```
Turnaround Time = Completion Time - Arrival Time
Waiting Time = Turnaround Time - Burst Time
CPU Utilization = (CPU Busy Time / Total Time) × 100%
Effective Access Time = (1-p) × Memory Access + p × Page Fault Time
Disk Access Time = Seek Time + Rotational Latency + Transfer Time
```

## Interview Tips

1. **Start with the basics** — define the concept clearly before diving into details
2. **Explain the "why"** — what problem does this concept solve?
3. **Give examples** — code snippets or real-world scenarios make concepts concrete
4. **Discuss trade-offs** — nothing is universally best; show you understand the alternatives
5. **Connect to your experience** — "In my C++ work, I've used..." makes answers memorable

## About These Notes

These notes are tailored for a C++ developer with 1.5 years of experience preparing for technical interviews. The content emphasizes:

- **Discussion-style explanations** that help you articulate concepts in interviews
- **C++ specific examples** showing how OS concepts relate to your daily work
- **Practical trade-offs** that demonstrate deeper understanding
- **Common interview questions** with detailed, conversational answers

Good luck with your interview!
