# Chapter 5: Deadlocks

## 5.1 What is a Deadlock?

A **deadlock** is a situation where a set of processes are blocked forever, each waiting for a resource held by another process in the set. No process can proceed because each is waiting for something that will never happen. The system doesn't crash — it just freezes, which makes deadlocks particularly insidious.

Consider a classic example with two mutexes. Thread 1 acquires mutex A and then tries to acquire mutex B. Meanwhile, Thread 2 acquires mutex B and then tries to acquire mutex A. Now Thread 1 holds A and waits for B, while Thread 2 holds B and waits for A. Neither can proceed — they're deadlocked.

```cpp
// Thread 1                    // Thread 2
mutex_A.lock();                mutex_B.lock();
// ... some work ...           // ... some work ...
mutex_B.lock();  // WAITS      mutex_A.lock();  // WAITS
```

This is one of the most critical issues in concurrent programming. Deadlocks often appear in production under load when timing conditions align just right. The key is understanding the four necessary conditions and designing systems to prevent at least one of them.

## 5.2 The Four Necessary Conditions (Coffman Conditions)

For a deadlock to occur, all four of these conditions must hold simultaneously. If we can prevent even one, deadlock is impossible.

**Mutual Exclusion** means that at least one resource must be held in a non-shareable mode. Only one process can use the resource at a time. For example, a printer can only print one job at a time, and a mutex can only be held by one thread. This condition is often inherent to the resource and cannot be eliminated.

**Hold and Wait** means that a process must be holding at least one resource while waiting to acquire additional resources held by other processes. For example, Thread 1 holds mutex A and waits for mutex B.

**No Preemption** means that resources cannot be forcibly taken away from a process. A process must voluntarily release its resources. You can't forcibly unlock a mutex held by another thread.

**Circular Wait** means there must be a circular chain of processes, where each process is waiting for a resource held by the next process in the chain. P1 waits for a resource held by P2, P2 waits for a resource held by P3, and P3 waits for a resource held by P1.

These four conditions are necessary AND sufficient for deadlock. In an interview, if asked how to prevent deadlock, go through each condition and explain which ones can be practically eliminated. Mutual exclusion is usually inherent to the resource. No preemption is hard for resources like mutexes. So we typically focus on preventing Hold-and-Wait or Circular Wait.

## 5.3 Deadlock Handling Strategies

There are four main approaches to dealing with deadlocks.

**Prevention** designs the system so that deadlock is structurally impossible. We ensure that at least one of the four necessary conditions can never hold. This approach has high overhead because it's restrictive, but it guarantees no deadlocks.

**Avoidance** dynamically checks if a resource allocation would lead to an unsafe state. Before granting a request, the system checks if it would still be possible for all processes to complete. This has medium overhead due to runtime checks.

**Detection and Recovery** allows deadlocks to happen, detects when they occur, and then recovers. This has low overhead during normal operation but requires a recovery mechanism.

**Ignorance**, also called the Ostrich Algorithm, simply pretends deadlocks don't happen. Most general-purpose operating systems like Linux and Windows use this approach. The overhead of prevention or avoidance isn't worth it for general use, and deadlocks are left to application developers to handle.

## 5.4 Deadlock Prevention

Prevention ensures that at least one of the four necessary conditions cannot hold.

**Preventing Mutual Exclusion** would mean making resources shareable. This is often impossible — you can't share a printer mid-job or let two threads hold the same mutex. It works for read-only files or read locks in reader-writer locks, but rarely for most resources.

**Preventing Hold and Wait** ensures a process never holds resources while waiting for others. One method is to request all resources at once — the process either gets everything or nothing. Another method is to release all currently held resources before requesting new ones. In C++, `std::scoped_lock` implements this by locking multiple mutexes atomically:

```cpp
std::scoped_lock lock(mutex_A, mutex_B);  // All or nothing
```

The problems with this approach are low resource utilization (resources are held but not used) and potential starvation (a process needing many popular resources may never get all of them). You also may not know all resources needed upfront.

**Preventing No Preemption** allows resources to be forcibly taken away. If a process holding resources requests another that can't be immediately allocated, it releases all currently held resources. This only works for resources whose state can be saved and restored, like CPU registers or memory pages. It doesn't work for mutexes or printers.

**Preventing Circular Wait** is the most practical approach. We impose a total ordering on resources and require processes to request resources in increasing order. Assign a unique number to each resource: R1=1, R2=2, R3=3. Processes must request resources in increasing order of their numbers.

```cpp
// Resource ordering: mutex_A = 1, mutex_B = 2
// CORRECT: Always lock in order A, then B
void thread1() {
    std::lock_guard<std::mutex> l1(mutex_A);  // 1
    std::lock_guard<std::mutex> l2(mutex_B);  // 2
}

void thread2() {
    std::lock_guard<std::mutex> l1(mutex_A);  // 1 (same order!)
    std::lock_guard<std::mutex> l2(mutex_B);  // 2
}
```

Why does this work? If all processes request in increasing order, you can't have P1 holding R1 and waiting for R2 while P2 holds R2 and waits for R1. P2 would have had to request R1 first, so it couldn't be holding R2 while waiting for R1.

## 5.5 Deadlock Avoidance

Instead of structurally preventing deadlock, avoidance dynamically checks each resource request to ensure the system stays in a **safe state**.

A **safe state** is one where there exists a sequence of processes such that each process can get its maximum required resources, finish, and release resources for the next process. An **unsafe state** is one where no such sequence exists. Being in an unsafe state doesn't mean deadlock has occurred, but it means deadlock is possible.

The **Banker's Algorithm** is the most famous deadlock avoidance algorithm, developed by Dijkstra. It's named because it models how a banker might allocate limited funds to customers.

The algorithm uses several data structures. For n processes and m resource types: `Available[m]` is the number of available instances of each resource type. `Max[n][m]` is the maximum demand of each process, declared upfront. `Allocation[n][m]` is the resources currently allocated to each process. `Need[n][m]` is the remaining resources each process might need, calculated as Max minus Allocation.

The safety algorithm determines if the current state is safe. It simulates processes completing and releasing resources. If we can find a sequence where all processes complete, the state is safe.

When a process requests resources, the algorithm first checks if the request is valid (not exceeding declared maximum and not exceeding available). Then it pretends to allocate the resources and runs the safety algorithm. If the resulting state is safe, the request is granted. If unsafe, the request is denied and the process must wait.

The Banker's Algorithm is elegant but has practical limitations. It requires knowing maximum resource needs upfront, which isn't always possible. It also has O(n²m) complexity for each request, which is expensive. That's why most real operating systems don't use it. However, understanding it demonstrates knowledge of safe states and resource allocation concepts.

## 5.6 Deadlock Detection

Instead of preventing or avoiding deadlock, we can allow it to happen and then detect and recover from it.

A **Resource Allocation Graph (RAG)** is a directed graph representing resource allocation and requests. Process nodes are drawn as circles, and resource nodes are drawn as rectangles with dots inside representing instances. A request edge goes from a process to a resource, indicating the process is waiting for that resource. An assignment edge goes from a resource to a process, indicating the resource is allocated to that process.

For single-instance resources, deadlock exists if and only if there's a cycle in the RAG. For multiple-instance resources, a cycle is necessary but not sufficient for deadlock — you need to run a detection algorithm similar to the Banker's safety algorithm.

When should we run detection? Running it on every resource request gives immediate detection but has high overhead. Running it periodically (every N minutes) or when CPU utilization drops below a threshold is more practical. Deadlock often causes low CPU usage because deadlocked processes aren't using CPU.

## 5.7 Deadlock Recovery

Once deadlock is detected, we must break it. There are two main approaches.

**Process Termination** can abort all deadlocked processes, which is simple and guaranteed to work but loses all their work. Alternatively, we can abort one process at a time, re-running detection after each abort until the deadlock is broken. Selection criteria include process priority, how long it has been running, how many resources it holds, how many more it needs, and whether it's interactive or batch.

**Resource Preemption** forcibly takes resources from some processes and gives them to others. The challenges are selecting a victim (which process to preempt), rollback (what state to return the preempted process to — either total rollback to the beginning or partial rollback to a checkpoint), and starvation (the same process might be selected as victim repeatedly, so we include the number of rollbacks in the cost factor).

Recovery is the ugly part of deadlock handling. Terminating processes loses work. Preemption requires rollback capability, which many resources don't support — you can't "un-print" a page. In practice, most systems either prevent deadlock through careful design or simply restart the affected application.

## 5.8 Deadlock in C++

In C++ code, the most practical approach is preventing circular wait through lock ordering.

**Lock Ordering** establishes a convention to always acquire locks in a consistent order, like alphabetically by name or by memory address:

```cpp
std::mutex m1, m2;

void safe_function() {
    std::lock_guard<std::mutex> l1(m1);  // Always m1 first
    std::lock_guard<std::mutex> l2(m2);  // Then m2
}
```

**std::scoped_lock** (C++17) handles multiple mutexes safely using a deadlock-avoidance algorithm internally:

```cpp
void safe_function() {
    std::scoped_lock lock(m1, m2);  // Locks both, deadlock-free
}
```

**std::lock** can lock multiple mutexes atomically before C++17:

```cpp
void safe_function() {
    std::lock(m1, m2);  // Lock both atomically
    std::lock_guard<std::mutex> l1(m1, std::adopt_lock);
    std::lock_guard<std::mutex> l2(m2, std::adopt_lock);
}
```

**Timeouts** provide another approach — try to acquire the lock for a limited time and back off if unsuccessful:

```cpp
std::unique_lock<std::mutex> lock(m, std::defer_lock);
if (lock.try_lock_for(std::chrono::seconds(1))) {
    // Got the lock
} else {
    // Timeout — handle gracefully, maybe retry later
}
```

## 5.9 Livelock and Starvation

Two related but different issues are livelock and starvation.

**Livelock** occurs when processes are not blocked but make no progress because they keep responding to each other. Imagine two people in a hallway trying to pass — both step left, then both step right, then both step left again, forever moving but never passing. In code, this can happen when threads keep backing off and retrying in sync. The solution is to add random backoff delays so they eventually get out of sync.

**Starvation** occurs when a process waits indefinitely because other processes keep getting resources first. The system is making progress (other processes run), but one process never gets its turn. This differs from deadlock where no progress is made at all. The solution is aging — gradually increase the priority of waiting processes so they eventually get served.

## 5.10 Common Interview Questions

**Q: What are the four necessary conditions for deadlock?**

Mutual exclusion — resources can't be shared. Hold and wait — process holds resources while waiting for more. No preemption — resources can't be forcibly taken. Circular wait — circular chain of processes waiting for each other. All four must hold simultaneously for deadlock.

**Q: How would you prevent deadlock in your code?**

I use resource ordering — always acquire locks in a consistent order. In C++, I use `std::scoped_lock` which handles multiple mutexes safely. I also minimize lock scope and avoid holding locks while doing I/O or calling external code.

**Q: Explain Banker's Algorithm.**

It's a deadlock avoidance algorithm. Before granting a resource request, it simulates the allocation and checks if the system remains in a safe state — meaning there's a sequence where all processes can complete. If safe, grant the request. If unsafe, deny and make the process wait. It requires knowing maximum resource needs upfront.

**Q: What's the difference between deadlock prevention and avoidance?**

Prevention ensures deadlock is structurally impossible by eliminating one of the four conditions at design time. Avoidance allows all conditions but dynamically checks each request to ensure the system stays safe. Prevention is more restrictive but has no runtime overhead. Avoidance is more flexible but requires runtime checks.

## Key Takeaways

Deadlock is a serious concurrency issue where processes are blocked forever, each waiting for resources held by others. Understanding the four necessary conditions — mutual exclusion, hold and wait, no preemption, and circular wait — is essential. Preventing any one of these conditions prevents deadlock.

In practice, preventing circular wait through lock ordering is the most common approach. Always acquire locks in a consistent order. In C++, use `std::scoped_lock` for multiple mutexes — it handles the ordering automatically using a deadlock-avoidance algorithm.

The Banker's Algorithm is theoretically important but rarely used in practice because it requires knowing maximum resource needs upfront. Most general-purpose operating systems simply ignore deadlocks and leave them to application developers.

Recovery from deadlock is ugly — you either terminate processes (losing work) or preempt resources (requiring rollback). Prevention is almost always better than cure when it comes to deadlocks.
