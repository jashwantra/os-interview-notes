# Chapter 1: Process Management

## 1.1 What is a Process?

A **process** is a program in execution. When you write a C++ program and compile it, you get an executable file sitting on disk — that's just a program. The moment you run it, the OS loads it into memory, allocates resources, and it becomes a process.

Think of it this way: a program is like a recipe written in a cookbook, while a process is the actual cooking happening in the kitchen with ingredients, utensils, and a chef actively working. The recipe itself doesn't do anything until someone starts following it.

A process consists of several components that work together. (Memory Layout of a process)
The **Code Section**, also called the Text section, contains the compiled instructions of your program. 
The **Data Section** holds global and static variables that persist throughout the program's lifetime. 
The **Heap** is where dynamically allocated memory lives — when you use `new` in C++, that memory comes from the heap. 
The **Stack** manages function calls, storing local variables, return addresses, and function parameters. 
Finally, the **Process Control Block (PCB)** is metadata that the OS maintains about the process, which we'll discuss in detail shortly.

## 1.2 Process vs Thread

This is a classic interview question, and understanding the distinction deeply will help you explain it conversationally.

A process is an **independent execution unit** with its own memory space. Each process has its own address space, which means one process cannot directly access another process's memory. Creating a process is considered **heavyweight** because the OS must allocate separate memory, set up file descriptors, and initialize various resources. Communication between processes requires **Inter-Process Communication (IPC)** mechanisms since they can't share memory directly. The advantage is that if one process crashes, other processes remain unaffected.

A thread, on the other hand, is a **lightweight unit of execution within a process**. All threads within a process share the same address space — they share code, data, and heap memory. However, each thread has its own stack and registers, including its own program counter and stack pointer. Creating a thread is much lighter than creating a process because the OS doesn't need to duplicate the entire memory space. Communication between threads is easier since they can directly access shared memory. The downside is that if one thread crashes with something like a segmentation fault, the entire process goes down.

When would you use each? In my C++ work, I use threads when I need concurrent execution within the same application — for example, one thread handling user input while another processes data in the background. Since threads share memory, I can easily pass data between them using shared variables, but this also means I need to be careful about synchronization using mutexes to prevent race conditions. If I needed complete isolation between tasks, or if I'm worried about one component crashing and affecting others, I'd use separate processes instead. Browsers often use this model — each tab runs as a separate process, so a crash in one tab doesn't bring down the entire browser.

## 1.3 Process Control Block (PCB)

The **Process Control Block** is a data structure maintained by the OS for every process. Think of it as the process's identity card that the OS uses to manage it. Without the PCB, the OS couldn't keep track of what each process is doing or resume it after a context switch.

The PCB contains several important pieces of information. The **Process ID (PID)** is a unique identifier for the process. The **Process State** indicates whether the process is running, waiting, ready, or in another state. The **Program Counter** stores the address of the next instruction to execute. The **CPU Registers** hold the contents of all process-centric registers at the time of the last context switch. **CPU Scheduling Information** includes the process priority and pointers to scheduling queues. **Memory Management Information** contains page tables, segment tables, and base/limit registers. **I/O Status Information** tracks the list of open files and I/O devices allocated to the process. Finally, **Accounting Information** records CPU time used, time limits, and other statistics.

The PCB is crucial for multitasking. When the OS switches from one process to another, it saves everything about the current process — the program counter, registers, memory mappings — into that process's PCB. When the OS comes back to that process later, it restores everything from the PCB, and the process continues as if nothing happened. This is the foundation of how modern operating systems can run many programs seemingly simultaneously.

## 1.4 Process States and Lifecycle

A process goes through several states during its lifetime. Understanding this state diagram is fundamental to understanding how the OS manages processes.

The five main states are **New**, **Ready**, **Running**, **Waiting** (also called Blocked), and **Terminated**. When a process is in the **New** state, it's being created — memory is being allocated and the PCB is being set up. Once initialization is complete, the process moves to the **Ready** state, meaning it's loaded in memory and waiting for CPU time. Multiple processes can be in the ready state simultaneously, sitting in what's called the ready queue.

When the scheduler selects a process from the ready queue, it moves to the **Running** state. On a single-core system, only one process can be running at any given moment. From the running state, several transitions are possible. If the process's time slice expires or a higher-priority process arrives, it gets preempted and moves back to Ready. If the process requests I/O or waits for some event, it moves to the **Waiting** state — it cannot proceed until that event occurs. When the I/O completes or the event happens, the process moves back to Ready. Finally, when the process finishes execution or calls `exit()`, it moves to the **Terminated** state, where the OS cleans up by deallocating memory, closing files, and removing the PCB.

Here's a practical example: when I call a blocking function like `read()` on a file in my C++ program, my process moves from Running to Waiting state. The OS then schedules another process to use the CPU. When the disk I/O completes, my process moves to Ready state and waits for the scheduler to pick it up again. This is why blocking I/O can hurt performance in single-threaded applications — the CPU sits idle waiting for I/O instead of doing useful work.

## 1.5 Context Switching

**Context switching** is the mechanism by which the OS saves the state of one process and loads the state of another, allowing multiple processes to share a single CPU. This is what enables multitasking.

When a context switch happens, several things occur in sequence. First, an interrupt or system call triggers the switch — this could be a timer interrupt indicating the time slice expired, an I/O interrupt, or a system call from the process itself. The OS then saves the current process's state, including all CPU registers, the program counter, and the stack pointer, into that process's PCB. The OS updates the PCB to reflect the new state (Running to Ready, or Running to Waiting). Next, the scheduler selects a process from the ready queue. The OS loads the new process's state from its PCB, restoring all the registers and the program counter. If the system uses virtual memory, the OS also updates the memory mappings by switching page tables. Finally, the CPU jumps to the instruction pointed to by the new program counter, and the new process resumes execution.

Context switching is **pure overhead** — during the switch, no useful work is done. The cost includes saving and restoring registers, flushing and reloading CPU caches (which causes cache pollution), flushing the TLB (Translation Lookaside Buffer), and flushing the CPU pipeline. On modern systems, a context switch typically takes 1-10 microseconds. This might seem small, but with thousands of switches per second, it adds up.

This overhead is why threads are often preferred over processes for tasks that need frequent switching. Thread context switches are cheaper because threads share the same address space, so there's no need to switch page tables or flush the TLB.

## 1.6 Inter-Process Communication (IPC)

Since processes have separate address spaces, they cannot directly share data. **Inter-Process Communication** provides mechanisms for processes to communicate and synchronize with each other.

Why do we need IPC? There are several reasons. **Data sharing** allows one process to produce data that another consumes. **Modularity** lets us break an application into cooperating processes. **Privilege separation** enables running sensitive operations in separate processes for security. **Parallelism** allows distributing work across multiple processes.

**Pipes** are one of the simplest IPC mechanisms. A pipe is a unidirectional communication channel where data flows in one direction — one process writes, another reads. Anonymous pipes work between parent-child processes, while named pipes (also called FIFOs) can work between unrelated processes. In C++, you'd create a pipe with the `pipe()` system call, which gives you two file descriptors — one for reading and one for writing.

**Message Queues** allow processes to send and receive messages through a queue maintained by the OS. Messages can have types, allowing selective receiving. This mechanism is asynchronous — the sender doesn't have to wait for the receiver to be ready.

**Shared Memory** is the fastest IPC mechanism because after the initial setup, there's no kernel involvement. A region of memory is mapped into multiple processes' address spaces, so they can read and write to it directly. However, this requires synchronization using semaphores or mutexes to prevent race conditions. Shared memory is ideal when large amounts of data need to be shared frequently.

**Sockets** enable communication over a network or locally through Unix domain sockets. They provide bidirectional communication and work across machines, making them essential for client-server architectures.

**Signals** are asynchronous notifications sent to a process. They carry limited information — just a signal number — so they're used for event notification rather than data transfer. Examples include `SIGKILL` to forcefully terminate a process, `SIGTERM` for graceful termination, and `SIGINT` which is sent when you press Ctrl+C.

In a project, if I had to choose between shared memory and message queues for IPC, I'd consider the trade-offs. Shared memory is faster since data doesn't copy through the kernel, but it requires careful synchronization with semaphores. Message queues are simpler because the OS handles the buffering and synchronization, but they have more overhead. For high-throughput scenarios, I'd go with shared memory; for simpler, less frequent communication, message queues are easier to maintain.

## 1.7 Special Process States: Zombies and Orphans

Two special process states often come up in interviews: zombie processes and orphan processes.

A **zombie process** is a process that has finished execution but still has an entry in the process table. This happens because the parent process hasn't called `wait()` to read the child's exit status. The process is dead — it's not executing and has released most of its resources — but it's not fully cleaned up. The OS keeps the entry around so the parent can retrieve the exit status. Too many zombies can exhaust the process table, preventing new processes from being created. The fix is simple: the parent must call `wait()` or `waitpid()` to reap the zombie.

An **orphan process** is a process whose parent has terminated before it did. When this happens, the OS reassigns the orphan to the `init` process (PID 1), which becomes its new parent. The orphan continues running normally — it's not affected by its parent's termination. The `init` process periodically calls `wait()` to clean up any orphans that terminate.

## 1.8 Common Interview Questions

**Q: What happens when you type `./myprogram` and press Enter?**

The shell calls `fork()` to create a child process. The child then calls `exec()` to replace its memory image with `myprogram`. The OS loader reads the executable, sets up the code, data, heap, and stack segments, initializes the PCB, and puts the process in the Ready queue. The scheduler eventually picks it up and it starts running.

**Q: Can a child process run after its parent terminates?**

Yes, it becomes an orphan process. The OS reassigns it to the `init` process (PID 1), which becomes its new parent. The orphan continues running normally.

**Q: Why is thread creation faster than process creation?**

Threads share the parent's address space, so the OS doesn't need to duplicate memory mappings, page tables, or allocate new memory regions. It only needs to create a new stack and thread control block. Process creation requires copying or setting up an entirely new address space.

**Q: When would you prefer multiple processes over multiple threads?**

When I need isolation — if one component crashes, I don't want it to bring down the entire application. Also, for security — processes have separate memory spaces, so a vulnerability in one process can't directly access another's memory. Browsers use this model — each tab is often a separate process.

## Key Takeaways

Understanding process management is fundamental to understanding how operating systems work. A process is more than just a running program — it's a complete execution environment with its own memory space, resources, and state information stored in the PCB. The distinction between processes and threads is crucial: processes provide isolation but are heavyweight, while threads are lightweight but share memory and can affect each other.

Context switching is the mechanism that enables multitasking, but it comes with overhead. The OS must save and restore state, which takes time and affects cache performance. This is why thread switches are cheaper than process switches.

IPC mechanisms exist because processes can't directly share memory. Each mechanism has trade-offs: shared memory is fastest but requires synchronization, while message queues are simpler but slower. Understanding these trade-offs helps you make good design decisions.

Finally, understanding process states and transitions helps you reason about program behavior. When your program blocks on I/O, it moves to the Waiting state, and the OS schedules another process. This is why asynchronous I/O and non-blocking operations can improve performance in I/O-bound applications.
