# Chapter 10: Interview Scenarios & C++ Specific Topics

## 10.1 Classic Interview Questions with Discussion-Style Answers

These are frequently asked questions in OS interviews. Practice explaining them conversationally, as if you're discussing with a colleague.

### What happens when you type `./program` and press Enter?

This is a favorite interview question because it touches multiple OS concepts. Here's how I'd explain it:

"When I type `./program` and press Enter, the shell process handles my input. The shell first calls `fork()` to create a child process. This child is initially a copy of the shell itself. The child then calls `exec()` — specifically one of the exec family like `execve()` — to replace its memory image with the new program.

The OS loader reads the executable file, which on Linux is typically in ELF format. It sets up the process's address space with several segments: the text segment containing the compiled code (marked read-only and executable), the data segment for initialized global variables, the BSS segment for uninitialized globals (which the OS zeroes out), and it allocates space for the stack and heap.

The OS sets up page tables for the new process, but physical memory isn't allocated yet — that happens on demand when pages are first accessed. If the program uses shared libraries like libc, the dynamic linker loads those and resolves symbols.

Finally, the OS jumps to the program's entry point. This isn't `main()` directly — it's typically a function called `_start` provided by the C runtime, which sets up the environment and then calls `main()`. Meanwhile, the parent shell calls `wait()` to block until the child terminates. When the program finishes and calls `exit()`, the OS cleans up its resources, and the shell resumes to show the next prompt."

### What happens when you type a URL in a browser?

This question tests knowledge of both OS and networking concepts. Here's a comprehensive answer:

"This involves several OS layers working together. First, the browser needs to resolve the domain name to an IP address. It checks its local DNS cache, then the OS cache, and if not found, makes DNS queries through system calls to contact DNS servers.

Once we have the IP address, the browser creates a socket using the `socket()` system call. It then calls `connect()` to initiate a TCP connection, which involves the three-way handshake — SYN, SYN-ACK, ACK. The OS network stack handles the TCP/IP protocol details.

The browser sends the HTTP request using `send()` or `write()` system calls. The OS buffers this data and handles transmission, including segmentation, checksums, and retransmissions if needed.

While waiting for the response, the process might block on `recv()`, or in modern browsers, it uses asynchronous I/O so the UI thread stays responsive. When data arrives, the network card generates an interrupt. The OS network stack reassembles TCP segments into the HTTP response.

The browser then parses the HTML, which might trigger more requests for CSS, JavaScript, and images — each following a similar process. Modern browsers spawn multiple threads for parsing, JavaScript execution, and rendering. Finally, the browser uses graphics system calls to render the page to the screen.

Throughout this process, the OS manages memory allocation for buffers, handles context switches between the browser and other processes, and mediates all hardware access through system calls."

### Process vs Thread — when would you use each?

"The key distinction is isolation versus communication efficiency. A process has its own address space — complete memory isolation. A thread shares the process's address space with other threads.

I'd use processes when I need isolation. If one component crashes, I don't want it to bring down the entire application. Browsers use this model — each tab is often a separate process, so a crash in one tab doesn't affect others. Processes are also better for security — a vulnerability in one process can't directly access another's memory.

I'd use threads when I need fast communication between concurrent tasks. Since threads share memory, they can communicate through shared variables without the overhead of IPC. Thread creation is also cheaper than process creation because there's no need to set up a new address space.

In my C++ work, I typically use threads for parallelism within an application — for example, one thread handling user input while another processes data in the background. I use `std::thread` or `std::async` for this. If I needed complete isolation between components, or if I'm building a service that should survive component failures, I'd use separate processes communicating via sockets or shared memory."

### Explain deadlock with a real-world example

"Deadlock is when two or more processes are blocked forever, each waiting for a resource held by another. The classic example is two threads transferring money between accounts:

```cpp
// Thread 1: Transfer from A to B
lock(account_A);
lock(account_B);  // Waits for Thread 2 to release B
// Transfer...

// Thread 2: Transfer from B to A
lock(account_B);
lock(account_A);  // Waits for Thread 1 to release A
// Transfer...
```

Thread 1 holds A and waits for B. Thread 2 holds B and waits for A. Neither can proceed — they're deadlocked.

Four conditions must hold for deadlock: mutual exclusion (only one thread can hold a lock), hold and wait (holding one resource while waiting for another), no preemption (can't forcibly take a lock), and circular wait (the circular dependency we see here).

In my code, I prevent deadlock primarily by preventing circular wait through lock ordering. I always acquire locks in a consistent order — for example, always lock the account with the lower ID first. In C++17, I use `std::scoped_lock` which handles multiple mutexes safely using a deadlock-avoidance algorithm internally."

### What is virtual memory and why is it useful?

"Virtual memory is an abstraction that gives each process the illusion of having its own large, contiguous address space, regardless of how much physical RAM is available or how it's actually laid out.

Here's how it works: each process has a page table that maps virtual addresses to physical addresses. The MMU (Memory Management Unit) translates addresses at runtime. Pages not currently in RAM are stored on disk and loaded on demand when accessed.

The benefits are substantial. First, isolation — Process A can't access Process B's memory because they have different page tables mapping to different physical locations. Second, we can use more memory than physically available — the OS swaps pages to disk as needed. Third, programming is simplified — I don't need to worry about where my data physically resides or coordinate with other programs. Fourth, efficient sharing — multiple processes can share code pages for common libraries like libc.

In C++, when I allocate a large array with `new`, I get virtual memory. Physical RAM is allocated only when I actually touch those pages. This is why I can allocate more than physical RAM — as long as I don't use it all at once. It also means allocating a large sparse data structure is efficient — only the pages I actually use consume physical memory."

### What is a race condition and how do you fix it?

"A race condition occurs when program behavior depends on the timing of thread execution — specifically when multiple threads access shared data and at least one modifies it without proper synchronization.

Here's a simple example:

```cpp
int counter = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++;  // Race condition!
    }
}
```

If two threads run this, you might expect counter to be 200000. But it likely won't be. The operation `counter++` involves three steps: read the current value, add one, write the new value. Two threads can interleave these steps, causing lost updates.

I fix race conditions through synchronization. For simple counters, I use `std::atomic<int>` which uses hardware atomic instructions. For more complex critical sections, I use `std::mutex` with `std::lock_guard` for RAII-style locking. I also try to minimize shared mutable state — if data doesn't need to be shared, make it thread-local; if it doesn't need to be mutable, make it const.

Detection is also important. I use ThreadSanitizer (`-fsanitize=thread`) during development to catch race conditions. Code review focusing on shared state is essential. And stress testing under load often reveals timing-dependent bugs."

## 10.2 C++ Memory Model and OS Relationship

Understanding how C++ memory relates to OS concepts is valuable for a C++ developer.

### Memory Layout

When your C++ program runs, its memory is organized into several regions. At the bottom (low addresses) is the **text segment** containing your compiled code — it's read-only and executable. Above that is the **data segment** for initialized global and static variables. The **BSS segment** holds uninitialized globals, which the OS zeroes. The **heap** grows upward from there when you use `new` or `malloc`. At the top (high addresses) is the **stack**, which grows downward as functions are called.

### How C++ Operations Map to OS Mechanisms

When you declare a local variable, it goes on the stack — no system call needed, just moving the stack pointer. When you call `new`, it allocates from the heap. The C++ runtime maintains a memory pool; small allocations come from this pool without a system call. Only when the pool is exhausted does it call `brk()` or `mmap()` to get more memory from the OS.

When you create a `std::thread`, it calls `clone()` on Linux to create a kernel thread. When you use `std::mutex`, it uses futex (fast userspace mutex) — in the uncontended case, it's just an atomic operation in userspace with no system call. Only when contention occurs does it make a system call to sleep.

`std::atomic` operations use CPU atomic instructions directly — no system call at all. File I/O with `std::fstream` eventually calls `open()`, `read()`, `write()` system calls, though the C++ library buffers data to minimize syscalls.

### Memory Allocation Layers

The allocation stack has several layers. Your `new`/`delete` calls go to the C++ runtime's `operator new`/`delete`. These call the C library's `malloc`/`free`, which maintains a memory pool. Only when the pool needs more memory does it call the OS via `brk()` or `mmap()`.

This layering is important for performance. System calls are expensive — about 100-1000 CPU cycles. By maintaining a pool, `malloc` can satisfy most allocations without a syscall. This is why custom allocators can improve performance — they can be optimized for specific allocation patterns.

## 10.3 C++ Threading and OS Threads

When you create a `std::thread` in C++, you're creating a kernel thread. On Linux, this uses the One-to-One model — each `std::thread` corresponds to one kernel thread, created via the `clone()` system call with the `CLONE_THREAD` flag. The kernel scheduler manages these threads just like processes.

**Thread Local Storage** with `thread_local` gives each thread its own copy of a variable. No synchronization is needed because each thread accesses its own copy. The OS provides this through segment registers or thread control blocks.

C++ synchronization primitives are optimized for the common case. `std::mutex` uses futex on Linux. In the uncontended case — when no other thread holds the lock — it's just an atomic compare-and-swap in userspace, no system call. Only when a thread needs to wait does it make a `futex()` syscall to sleep. This is why mutexes are efficient for short critical sections with low contention.

## 10.4 Memory Ordering

Modern CPUs reorder memory operations for performance. This can break multi-threaded code if you're not careful.

Consider this example:

```cpp
int data = 0;
bool ready = false;

// Thread 1
data = 42;
ready = true;

// Thread 2
if (ready)
    use(data);  // Might see data = 0!
```

The CPU might reorder `ready = true` before `data = 42` because there's no dependency between them from a single-threaded perspective. Thread 2 might see `ready` as true but `data` still as 0.

C++ provides memory orders to control this. `memory_order_relaxed` provides only atomicity — no ordering guarantees. `memory_order_acquire` ensures no reads/writes can be reordered before this operation. `memory_order_release` ensures no reads/writes can be reordered after this operation. `memory_order_seq_cst` (the default) provides full sequential consistency.

For the example above, the fix is:

```cpp
std::atomic<bool> ready{false};

// Thread 1
data = 42;
ready.store(true, std::memory_order_release);

// Thread 2
while (!ready.load(std::memory_order_acquire));
use(data);  // Guaranteed to see 42
```

The release-acquire pair creates a synchronization point. All writes before the release are visible after the acquire.

## 10.5 Real-World Scenarios

### High-Performance Logger

A high-performance logger needs to minimize the impact on the main application. The key techniques are using a lock-free ring buffer for the log entries, having a dedicated writer thread that batches writes to disk, and using `std::atomic` for the write index to avoid mutex overhead.

The OS concepts involved include lock-free data structures using atomic operations, dedicated I/O threads to avoid blocking the main thread, and batched writes to minimize system call overhead.

### Memory Pool Allocator

When you have many small allocations of the same size, a memory pool is much more efficient than calling `new` each time. You allocate a large block via `mmap()` once, then manage it yourself with a free list.

The OS concepts involved include reducing system calls by pre-allocating, avoiding fragmentation with fixed-size blocks, and improving cache locality by allocating from contiguous memory.

### Memory-Mapped Files

For processing large files, memory mapping is often more efficient than explicit read/write calls:

```cpp
int fd = open("data.bin", O_RDONLY);
char* data = (char*)mmap(nullptr, size, PROT_READ, MAP_PRIVATE, fd, 0);

// Access file as if it were an array
for (size_t i = 0; i < size; i++)
    process(data[i]);

munmap(data, size);
```

The OS handles paging automatically. Pages are loaded on demand when accessed. The OS can prefetch sequential pages. The file stays in the page cache for future access. There are no explicit `read()` system calls.

## 10.6 Interview Tips

When answering OS questions in an interview, start with the basics and define the concept clearly. Explain the "why" — what problem does this solve? Give examples, either code or real-world scenarios. Discuss trade-offs because nothing is universally best. Connect to your experience with phrases like "In my C++ work, I've used..."

Common mistakes to avoid include memorizing without understanding (interviewers can tell), being vague (use specific terms like "mutex" not "lock thing"), forgetting trade-offs (every design has pros and cons), and panicking on unknowns (say "I'm not sure, but I'd approach it by...").

## Key Takeaways

Understanding OS concepts makes you a better C++ developer. When you know that `new` doesn't always make a system call because of memory pools, you understand why custom allocators can help. When you know that `std::thread` creates a kernel thread, you understand the overhead of thread creation and why thread pools matter.

The key concepts to remember are: processes provide isolation while threads share memory; virtual memory provides the illusion of large contiguous address space; synchronization is essential for shared mutable state; system calls have overhead so batching and buffering help; and memory ordering matters for lock-free code.

In interviews, demonstrate that you understand not just what these concepts are, but why they exist and how they affect real code. Connect OS concepts to your C++ experience. Show that you can reason about trade-offs and make informed design decisions.

Good luck with your interview!
