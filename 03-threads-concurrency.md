# Chapter 3: Threads & Concurrency

## 3.1 What is a Thread?

A **thread** is the smallest unit of execution within a process. While a process is a program in execution with its own memory space, a thread is a single sequence of instructions executing within that process. You can think of a process as a factory with its own building, equipment, and resources, while threads are the workers inside that factory — they share the factory's resources but each worker has their own task list and workspace.

Within a process, all threads share certain resources. They share the **code section**, meaning they all execute the same program instructions. They share the **data section**, which contains global and static variables. They share the **heap**, where dynamically allocated memory lives. They also share open files and signals. However, each thread has its own **thread ID** to uniquely identify it, its own **program counter** indicating where in the code this thread is currently executing, its own **register set** containing CPU register values, and its own **stack** for local variables and function call history.

## 3.2 Why Use Threads?

There are several compelling reasons to use threads in your applications.

**Responsiveness** is a major benefit. In a GUI application, you can have one thread handling user input while another thread performs heavy computation in the background. The UI remains responsive even during intensive operations. Without threads, the UI would freeze while the computation runs.

**Resource sharing** is another advantage. Since threads share the process's memory, they can easily communicate by reading and writing shared variables. This is much simpler and faster than inter-process communication mechanisms like pipes or message queues. Of course, this also means you need to be careful about synchronization.

**Economy** refers to the fact that creating a thread is much cheaper than creating a process. When you create a process, the OS must allocate a new address space, copy or set up memory mappings, and initialize various resources. When you create a thread, the OS only needs to allocate a new stack and set up a thread control block — the address space is already there.

**Scalability** means that threads can run in parallel on multi-core CPUs. If you have a quad-core processor, four threads can truly execute simultaneously, potentially giving you a 4x speedup for parallelizable work. A single-threaded program can only use one core at a time.

## 3.3 User-Level vs Kernel-Level Threads

This is a fundamental distinction in how threads are implemented and managed.

**User-Level Threads** are managed entirely in user space by a thread library. The kernel has no knowledge of these threads — it sees only the process. The thread library handles thread creation, scheduling, and synchronization. Context switching between user threads doesn't involve the kernel, making it very fast. User-level threads are also portable since the thread library can run on any OS.

However, user-level threads have significant disadvantages. The biggest problem is blocking. If one thread makes a blocking system call like a disk read, the entire process blocks because the kernel doesn't know about the other threads. There's also no true parallelism — since the kernel sees only one process, it runs on only one CPU even on a multi-core system. If one thread causes a page fault, all threads in the process wait.

**Kernel-Level Threads** are managed directly by the operating system kernel. The kernel is aware of each thread and schedules them individually. Thread creation, scheduling, and synchronization are system calls.

The advantages of kernel threads are significant. You get true parallelism because different threads can run on different CPUs simultaneously. If one thread blocks on I/O, the kernel can schedule another thread from the same process. This is essential for taking advantage of multi-core processors.

The disadvantages are that context switching is slower because it requires a kernel mode switch, and thread operations have more overhead since they're system calls. However, on modern systems, this overhead is acceptable, and the benefits of true parallelism far outweigh the costs.

Modern systems like Linux use kernel-level threads because we need true parallelism on multi-core CPUs. User-level threads were popular when systems had single cores and kernel thread overhead was significant. Today, the blocking problem of user-level threads is a bigger issue than the overhead of kernel threads.

## 3.4 Multithreading Models

These models describe how user-level threads map to kernel-level threads. Understanding these models is crucial for understanding how threading libraries work.

The **Many-to-One** model maps many user threads to one kernel thread. Thread management is done in user space, which is efficient. However, only one thread can access the kernel at a time, so if one thread blocks, all block. You cannot run threads in parallel on multiple cores because the kernel sees only one schedulable entity. Early versions of Solaris Green Threads used this model.

The **One-to-One** model maps each user thread to a corresponding kernel thread. This is what most modern systems use, including Linux with NPTL (Native POSIX Thread Library) and Windows. Each user thread has a corresponding kernel thread, giving you true parallelism — threads can run on different cores. If one thread blocks, others continue. The downside is more overhead since creating a user thread creates a kernel thread, and there may be limits on the number of kernel threads. This is why thread pools are important — you shouldn't create thousands of threads.

The **Many-to-Many** model multiplexes many user threads onto a smaller or equal number of kernel threads. This tries to get the best of both worlds: true parallelism plus efficient thread management. If one thread blocks, the kernel can schedule another user thread on that kernel thread. However, it's complex to implement correctly. Older versions of Solaris used this model, but most modern systems have moved to One-to-One because kernel thread overhead has decreased and the simplicity is worth it.

## 3.5 Thread Pools

Creating and destroying threads has overhead. Every time you create a thread, the OS must allocate memory for the stack, set up data structures, and potentially make system calls. If you're handling many short tasks, this overhead can exceed the actual work time. **Thread pools** solve this by maintaining a pool of pre-created threads ready to execute tasks.

Here's how a thread pool works. At initialization, you create N worker threads. When a task needs to be executed, it's added to a task queue. Worker threads pick tasks from the queue and execute them. After completing a task, the thread picks the next task from the queue rather than terminating. When the pool is no longer needed, threads are destroyed.

The benefits are substantial. You get reduced overhead because threads are reused, not created and destroyed for each task. You get bounded resources because the fixed number of threads prevents resource exhaustion — if a traffic spike hits your web server, extra requests wait in the queue rather than spawning unlimited threads. You get improved response time because threads are already created and ready. You get better throughput because the optimal number of threads can be tuned for your hardware.

Choosing the pool size depends on your workload. For CPU-bound tasks, the pool size should be approximately equal to the number of CPU cores. More threads would just cause context switching overhead without improving throughput. For I/O-bound tasks, you can have more threads since they spend time waiting for I/O. A common formula is: optimal threads = cores × (1 + wait_time/compute_time). In practice, you profile and tune based on actual performance.

## 3.6 C++ Threading

As a C++ developer, you should be comfortable with the standard threading facilities introduced in C++11.

To create a thread, you construct a `std::thread` object with a callable and its arguments:

```cpp
#include <thread>
#include <iostream>

void worker(int id) {
    std::cout << "Thread " << id << " running\n";
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

The `join()` method makes the calling thread wait for the thread to complete. The `detach()` method lets the thread run independently — its resources will be freed when it finishes. You must call one of these before the `std::thread` object is destroyed, otherwise `std::terminate()` is called and your program aborts. This is a safety feature to prevent accidentally orphaning threads or forgetting to wait for important work.

C++20 introduced `std::jthread`, which solves two common problems. First, it automatically joins in the destructor, so you can't accidentally forget to join and cause `std::terminate()`. Second, it provides cooperative cancellation through `stop_token`. Instead of forcefully killing a thread, you request it to stop, and the thread checks this token and exits gracefully:

```cpp
void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        // Do work
    }
    // Clean exit
}

std::jthread t(worker);
t.request_stop();  // Thread will check and exit gracefully
```

For higher-level asynchronous operations, `std::async` provides a cleaner abstraction:

```cpp
#include <future>

int compute(int x) { return x * x; }

auto future = std::async(std::launch::async, compute, 5);
// Do other work...
int result = future.get();  // Blocks until result is ready
```

The `std::async` function returns a `std::future` that will eventually hold the result. You can do other work and then call `get()` when you need the result. Be aware that the default launch policy might defer execution until `get()` is called, which isn't true parallelism. Use `std::launch::async` explicitly when you need guaranteed parallel execution.

## 3.7 Thread Safety

When multiple threads access shared data, you need synchronization. A **race condition** occurs when program behavior depends on the timing of thread execution.

Consider this simple example:

```cpp
int counter = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++;
    }
}
```

If two threads run this function, you might expect counter to be 200000. But it likely won't be. The operation `counter++` involves three steps: read the current value, add one, write the new value. Two threads can interleave these steps, causing lost updates.

The solutions involve synchronization. Using a mutex ensures only one thread accesses the counter at a time:

```cpp
std::mutex mtx;

void safe_increment() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        counter++;
    }
}
```

For simple types like counters, `std::atomic` is more efficient because it uses hardware atomic instructions without locking:

```cpp
std::atomic<int> counter{0};

void safe_increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++;  // Atomic increment
    }
}
```

Any time you have shared mutable state accessed by multiple threads, you need synchronization. The simplest approach is a mutex, but it can be a bottleneck. For simple counters and flags, `std::atomic` is more efficient. For complex data structures, consider lock-free algorithms or redesigning to minimize shared state.

## 3.8 Common Interview Questions

**Q: What's the difference between concurrency and parallelism?**

Concurrency is about dealing with multiple things at once — structuring a program to handle multiple tasks that can make progress independently. Parallelism is about doing multiple things at once — actually executing multiple tasks simultaneously on multiple cores. You can have concurrency without parallelism (single-core with time slicing) and parallelism without concurrency (SIMD operations on data).

**Q: When would you use processes instead of threads?**

I'd use processes when I need isolation — if one component crashes, I don't want it to bring down the whole application. Also for security — processes have separate address spaces, so a vulnerability in one can't directly access another's memory. Browsers use this model — each tab is often a separate process. Threads are better when I need fast communication and shared state.

**Q: What happens if you don't join or detach a thread in C++?**

If a `std::thread` object is destroyed without calling `join()` or `detach()`, `std::terminate()` is called and the program aborts. This is a safety feature to prevent accidentally orphaning threads or forgetting to wait for important work. C++20's `std::jthread` solves this by automatically joining in the destructor.

**Q: How many threads should a thread pool have?**

It depends on the workload. For CPU-bound tasks, roughly equal to the number of CPU cores — more threads would just cause context switching overhead. For I/O-bound tasks, you can have more threads since they spend time waiting. A common formula is cores × (1 + wait_time/compute_time). In practice, I profile and tune based on actual performance.

## Key Takeaways

Understanding threads is essential for modern software development. A thread is a lightweight execution unit within a process, sharing memory but having its own stack and registers. This sharing makes communication easy but requires careful synchronization.

Modern systems use kernel-level threads with a One-to-One model because we need true parallelism on multi-core CPUs. The overhead of kernel threads is acceptable compared to the benefits.

Thread pools are essential for production systems. Creating threads on demand is expensive and can exhaust resources. A pool of pre-created threads provides bounded concurrency and better performance.

In C++, always use RAII for thread management. `std::lock_guard` ensures mutexes are released even if exceptions occur. `std::jthread` in C++20 ensures threads are joined even if you forget. Race conditions are among the hardest bugs to debug, so identify shared mutable state and protect it properly.
