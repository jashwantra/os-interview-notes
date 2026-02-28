# Chapter 9: System Calls & Kernel Architecture

## 9.1 User Mode vs Kernel Mode

Modern operating systems use a **dual-mode operation** to protect the system from errant or malicious programs. The CPU has at least two privilege levels: user mode and kernel mode.

In **User Mode** (Ring 3 on x86), programs run with restricted privileges. They cannot directly access hardware, cannot access memory belonging to other processes or the kernel, and cannot execute privileged instructions. This is where your C++ applications run. If a user program tries to do something it's not allowed to, the CPU generates a trap, and the OS handles it — typically by terminating the offending process.

In **Kernel Mode** (Ring 0 on x86), the OS kernel runs with full privileges. It can access all hardware, all memory, and execute any instruction. The kernel is trusted code that manages resources and provides services to user programs.

The CPU has a **mode bit** that indicates the current privilege level. Transitions from user mode to kernel mode happen through well-defined mechanisms: system calls (when a program requests a service), interrupts (when hardware needs attention), or exceptions (when an error occurs, like division by zero). Transitions back to user mode happen when the kernel finishes handling the request.

**Privileged instructions** can only be executed in kernel mode. These include I/O instructions that access hardware directly, the halt instruction that stops the CPU, instructions that enable or disable interrupts, instructions that modify page tables or control registers, and instructions that change the mode bit itself. If a user program attempts any of these, the CPU traps to the kernel, which typically terminates the program with a segmentation fault or similar error.

This protection is fundamental to system stability. Without it, any buggy or malicious program could crash the entire system or access other users' data.

## 9.2 System Calls

A **system call** is the programmatic way a user program requests a service from the operating system kernel. It's the controlled gateway between user mode and kernel mode.

When your C++ program calls `open()` to open a file, it can't directly access the disk — that would require privileged I/O instructions. Instead, it makes a system call, asking the kernel to open the file on its behalf. The kernel validates the request, performs the operation, and returns the result.

The system call mechanism works as follows. First, the user program sets up parameters for the call, typically in CPU registers. The program then executes a special trap instruction — on x86-64 Linux, this is the `syscall` instruction; on older x86, it was `int 0x80`. This instruction causes the CPU to switch to kernel mode and jump to a predefined kernel entry point. The kernel saves the user program's context (registers, stack pointer), looks up the appropriate handler based on the system call number, and executes the handler. The handler performs the requested operation, places the return value in a register, restores the user context, and returns to user mode.

Parameters can be passed in several ways. Using **registers** is the most common and fastest method, but the number of registers is limited. Using the **stack** allows more parameters but is slower. Using a **memory block** where a register holds the address of a parameter block is useful for complex structures.

System calls have **overhead**. The mode switch itself takes time. The kernel must save and restore user context. TLB and cache effects can slow things down. A typical system call takes 100-1000 CPU cycles, compared to a regular function call which takes just a few cycles.

To minimize this overhead, programs use several strategies. **Buffered I/O** batches many small writes into fewer large writes — `printf` buffers output and only makes a system call when the buffer is full. **Memory-mapped files** avoid read/write system calls by mapping the file into memory. **vDSO (virtual Dynamic Shared Object)** on Linux provides user-space implementations of simple system calls like `gettimeofday()` that don't actually need kernel involvement.

## 9.3 Types of System Calls

System calls can be categorized by their purpose.

**Process Control** system calls manage processes. `fork()` creates a new process by duplicating the calling process. `exec()` replaces the current process image with a new program. `exit()` terminates the process and returns a status code. `wait()` makes a parent wait for a child process to terminate. `getpid()` returns the process ID.

**File Management** system calls handle files. `open()` opens a file and returns a file descriptor. `close()` closes a file descriptor. `read()` reads data from a file into a buffer. `write()` writes data from a buffer to a file. `lseek()` moves the file position pointer.

**Device Management** system calls control devices. `ioctl()` performs device-specific control operations. `mmap()` can map device memory into the process address space.

**Information Maintenance** system calls get or set system information. `time()` gets the current time. `getuid()` gets the user ID. `uname()` gets system information like the kernel version.

**Communication** system calls enable inter-process communication. `pipe()` creates a unidirectional communication channel. `socket()` creates a network socket. `shmget()` creates or accesses shared memory.

**Memory Management** system calls control memory. `brk()` and `sbrk()` change the size of the heap. `mmap()` maps files or anonymous memory into the address space. `mprotect()` changes memory protection flags.

In C++, you rarely call system calls directly. The C library provides wrapper functions that handle the details. When you call `fopen()`, it eventually calls the `open()` system call. When you use `new`, it eventually calls `brk()` or `mmap()` to get memory from the OS.

## 9.4 Kernel Architectures

The kernel architecture determines how OS services are organized and how they interact. There are three main approaches.

A **Monolithic Kernel** runs all OS services in kernel space as a single large program. The file system, device drivers, networking stack, and process management all run in kernel mode with full privileges. They communicate through direct function calls, which is very fast.

Linux, traditional Unix, and BSD use monolithic kernels. The advantages are excellent performance because there's no overhead for communication between components, and the design is straightforward — everything can call everything else directly. The disadvantages are that a bug in any component can crash the entire system, the kernel is large and complex, and the attack surface is large because all code runs with full privileges.

A **Microkernel** runs only the most essential services in kernel space — typically just inter-process communication (IPC), basic scheduling, and memory management. Everything else — file systems, device drivers, networking — runs as user-space processes called servers. These servers communicate through IPC.

Examples include Mach, L4, QNX, MINIX, and seL4. The advantages are reliability because a bug in a driver crashes only that driver, not the whole system. Security is better because the attack surface is minimal. The system is flexible because servers can be restarted without rebooting. The disadvantages are performance overhead from IPC between servers and increased complexity in the IPC mechanisms.

A **Hybrid Kernel** combines aspects of both. Core services that need high performance run in kernel space, while less critical services can run in user space. The kernel is modular but not as minimal as a microkernel.

Windows NT (and all subsequent Windows versions) and macOS (XNU) use hybrid kernels. They get good performance for critical paths while maintaining some modularity.

The debate between monolithic and microkernel architectures was fierce in the 1990s. Linus Torvalds famously argued for monolithic kernels, while Andrew Tanenbaum advocated for microkernels. In practice, monolithic kernels won for general-purpose operating systems because performance matters and the reliability benefits of microkernels weren't compelling enough. However, microkernels are used in specialized domains like real-time systems (QNX) and high-assurance systems (seL4).

## 9.5 Kernel Modules

**Loadable Kernel Modules (LKMs)** add flexibility to monolithic kernels. They allow code to be loaded into and unloaded from the kernel at runtime without rebooting.

Device drivers are the most common use case. When you plug in a USB device, the kernel loads the appropriate driver module. When you remove the device, the module can be unloaded. File systems can also be modules — you can add support for a new file system without recompiling the kernel.

On Linux, you can manage modules with commands like `lsmod` to list loaded modules, `modprobe driver_name` to load a module, and `modprobe -r driver_name` to remove a module.

It's important to understand that modules still run in kernel space with full privileges. They're not isolated like microkernel servers. A buggy module can still crash the system. Modules are about convenience and flexibility, not security or reliability.

## 9.6 Real-World Kernels

**Linux** uses a monolithic kernel with loadable modules. It's the most widely used kernel, running on everything from smartphones (Android) to supercomputers. It uses the Completely Fair Scheduler (CFS) for process scheduling and NPTL (Native POSIX Thread Library) for threading. Despite being monolithic, it's highly modular — most drivers are modules that can be loaded on demand.

**Windows NT** (the kernel used in Windows 10, 11, Server, etc.) uses a hybrid architecture. The kernel proper handles scheduling, interrupts, and synchronization. The Executive layer provides higher-level services like I/O management, object management, and security. The Hardware Abstraction Layer (HAL) isolates the kernel from hardware specifics. Device drivers run in kernel mode but are separate from the kernel itself.

**macOS XNU** is a hybrid combining the Mach microkernel with BSD. Mach provides the low-level primitives — IPC, memory management, scheduling. BSD provides the POSIX API, file systems, and networking. The I/O Kit handles device drivers. This combination gives macOS the IPC capabilities of Mach with the familiar Unix interface of BSD.

## 9.7 Common Interview Questions

**Q: Why do we need dual-mode operation?**

To protect the system from user programs. Without it, any program could access hardware directly, modify other programs' memory, or crash the system. Dual-mode ensures that only the trusted kernel can perform privileged operations, while user programs must request services through controlled system calls.

**Q: What happens when a system call is made?**

The program sets up parameters in registers and executes a trap instruction. The CPU switches to kernel mode and jumps to the kernel entry point. The kernel saves the user context, identifies the requested service, executes the handler, places the return value in a register, restores the user context, and returns to user mode.

**Q: What's the difference between monolithic and microkernel architectures?**

In a monolithic kernel, all OS services run in kernel space with full privileges, communicating through direct function calls. It's fast but a bug anywhere can crash the system. In a microkernel, only essential services run in kernel space; everything else runs as user-space servers communicating through IPC. It's more reliable and secure but has IPC overhead.

**Q: Why do most general-purpose OS use monolithic kernels?**

Performance. The IPC overhead of microkernels is significant for operations that happen frequently, like file I/O. The reliability benefits of microkernels haven't been compelling enough to outweigh the performance cost for general-purpose use. However, microkernels are used in specialized domains where reliability is critical.

## Key Takeaways

Dual-mode operation is fundamental to OS security. User programs run with restricted privileges and must use system calls to request kernel services. This controlled interface protects the system from buggy or malicious programs.

System calls are the gateway between user mode and kernel mode. They have overhead — typically 100-1000 cycles — so programs use buffering and batching to minimize the number of calls.

Kernel architecture involves trade-offs. Monolithic kernels are fast but risky — a bug anywhere crashes everything. Microkernels are safer but slower due to IPC overhead. Hybrid kernels try to balance both. In practice, monolithic kernels dominate general-purpose computing because performance matters.

Loadable kernel modules add flexibility to monolithic kernels, allowing drivers and file systems to be loaded at runtime. But they still run in kernel space with full privileges — they're about convenience, not isolation.

Understanding these concepts helps you reason about system behavior, performance, and security. When you call a function in C++, knowing whether it's a library function or a system call helps you understand its cost and behavior.
