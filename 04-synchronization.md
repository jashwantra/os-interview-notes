# Chapter 4: Synchronization

## 4.1 Why Synchronization?

When multiple threads or processes access shared resources concurrently, we need mechanisms to coordinate their access. Without proper synchronization, we get unpredictable behavior, corrupted data, and bugs that are notoriously hard to reproduce.

Consider a simple bank account scenario. You have a balance of 1000 rupees, and two threads are trying to withdraw money simultaneously — one wants to withdraw 500 and another wants to withdraw 800. Each thread first checks if there's enough balance, then performs the withdrawal. If both threads check the balance at the same time, both see 1000 and proceed with their withdrawals. The result? Your balance goes negative, which should never happen.

This is the fundamental problem that synchronization solves. We need ways to ensure that certain operations happen atomically — as indivisible units — or that only one thread can access shared data at a time.

## 4.2 Race Conditions

A **race condition** occurs when the behavior of a program depends on the relative timing of events, such as the order in which threads are scheduled. The results are non-deterministic — different runs may produce different results. These bugs are hard to reproduce because they depend on timing, and they often appear under load when context switches are more frequent.

There are two common types of race conditions. The **check-then-act** pattern is when you check a condition and then act based on it, but the condition changes between the check and the action. For example, checking if a file exists and then opening it — another process might delete the file between your check and your open. The **read-modify-write** pattern is when you read a value, modify it, and write it back — another thread might modify the value between your read and your write.

The classic example is incrementing a counter. The operation `counter++` looks atomic in code, but it actually involves three steps: read the current value from memory, add one to it, and write the new value back to memory. Two threads can interleave these steps, causing lost updates.

## 4.3 The Critical Section Problem

A **critical section** is a segment of code that accesses shared resources and must not be executed by more than one thread at a time. The critical section problem is about designing a protocol that processes can use to cooperate safely.

Any valid solution must satisfy three requirements. **Mutual Exclusion** means that if one process is executing in its critical section, no other process can be in its critical section. This is the fundamental requirement — only one at a time. **Progress** means that if no process is in the critical section and some processes want to enter, the selection of which process enters next cannot be postponed indefinitely. In other words, the mechanism itself shouldn't cause deadlock. **Bounded Waiting** means there must be a limit on the number of times other processes can enter their critical sections after a process has requested entry. This prevents starvation — every process should eventually get in.

When evaluating any synchronization mechanism, check these three properties. Mutual exclusion is obvious — that's the whole point. Progress means the mechanism itself doesn't cause deadlock. Bounded waiting prevents starvation — a process shouldn't wait forever while others keep jumping ahead.

## 4.4 Mutex (Mutual Exclusion Lock)

A **mutex** is a synchronization primitive that provides mutual exclusion. Only one thread can hold the mutex at a time. The two basic operations are `lock()` (or acquire), which takes the mutex if it's free or blocks until it becomes free, and `unlock()` (or release), which releases the mutex and allows a waiting thread to acquire it.

The key insight about mutexes is that they must be implemented atomically — the check-and-set operation must be indivisible. Modern CPUs provide atomic instructions like compare-and-swap (CAS) for this purpose.

A critical problem with manual lock/unlock is exception safety. Consider this code:

```cpp
mtx.lock();
do_something();  // What if this throws an exception?
mtx.unlock();    // This line is never reached!
```

If `do_something()` throws an exception, the mutex stays locked forever, causing a deadlock. The solution is RAII (Resource Acquisition Is Initialization) — use wrapper objects like `std::lock_guard` that automatically release the mutex when they go out of scope, even if an exception is thrown.

## 4.5 Semaphores

A **semaphore** is a more general synchronization primitive than a mutex. It maintains a counter and provides two atomic operations. The `wait()` operation (also called P, down, or acquire) decrements the counter if it's greater than zero, otherwise it blocks until the counter becomes positive. The `signal()` operation (also called V, up, or release) increments the counter and wakes up a waiting thread if any.

A **binary semaphore** has a counter that can only be 0 or 1, making it functionally similar to a mutex. A **counting semaphore** can have any non-negative integer value and is used to control access to a resource with multiple instances.

Consider a database connection pool with 5 connections. You'd initialize a semaphore with value 5. Each thread that wants a connection calls `wait()`, which decrements the counter. If all 5 connections are in use (counter is 0), the next thread blocks. When a thread finishes with a connection, it calls `signal()`, incrementing the counter and potentially waking a waiting thread.

The key difference between a semaphore and a mutex is ownership. A mutex must be unlocked by the thread that locked it — there's a concept of ownership. A semaphore has no ownership — any thread can call `signal()`. This makes semaphores useful for signaling between threads, not just mutual exclusion.

## 4.6 Monitors

A **monitor** is a high-level synchronization construct that encapsulates shared data and the operations on it. Only one thread can be active inside a monitor at a time — mutual exclusion is automatic and built into the construct.

Monitors include **condition variables** for threads to wait for specific conditions. The `wait()` operation releases the monitor lock, puts the thread to sleep, and reacquires the lock when woken. The `signal()` operation wakes up one waiting thread. The `broadcast()` operation wakes up all waiting threads.

There's an important subtlety about condition variables. When a thread is signaled and wakes up, it doesn't immediately run — it's moved to the ready queue and must compete for the monitor lock. By the time it actually runs, the condition it was waiting for might no longer be true. This is why you must always use a `while` loop to check conditions, not an `if` statement:

```cpp
// Correct: while loop
while (buffer.empty()) {
    cv.wait(lock);
}

// Incorrect: if statement
if (buffer.empty()) {
    cv.wait(lock);  // Condition might be false when we wake up!
}
```

This pattern handles both spurious wakeups (where a thread wakes up without being signaled) and the race between being signaled and actually running.

## 4.7 Spinlocks

A **spinlock** is a lock where the waiting thread busy-waits (spins) instead of sleeping. Instead of blocking and giving up the CPU, the thread continuously checks if the lock is available.

```cpp
class Spinlock {
    std::atomic<bool> locked{false};
public:
    void lock() {
        while (locked.exchange(true)) {
            // Spin - keep trying
        }
    }
    void unlock() {
        locked = false;
    }
};
```

Spinlocks are useful when the critical section is very short — just a few instructions. In this case, the overhead of putting a thread to sleep and waking it up might exceed the time spent spinning. They're also useful on multi-processor systems where one CPU can spin while another CPU releases the lock.

However, spinlocks are wasteful on single-processor systems. While one thread spins, the thread holding the lock can't run to release it — you're wasting CPU cycles. They're also bad for long critical sections or when holding the lock while doing I/O.

Modern implementations often use adaptive spinning — spin briefly, then sleep if the lock isn't released quickly.

## 4.8 C++ Synchronization Primitives

In modern C++, you have several synchronization primitives available.

`std::mutex` is the basic mutual exclusion primitive. You can call `lock()` and `unlock()` directly, but this is error-prone. Instead, use RAII wrappers.

`std::lock_guard` is the simplest RAII wrapper. It locks the mutex in its constructor and unlocks in its destructor. It's exception-safe and prevents forgetting to unlock:

```cpp
void safe_function() {
    std::lock_guard<std::mutex> lock(mtx);
    // Critical section
    // Automatically unlocked when lock goes out of scope
}
```

`std::unique_lock` is more flexible than `lock_guard`. You can unlock and relock it, defer locking, and use it with condition variables:

```cpp
std::unique_lock<std::mutex> lock(mtx);
// Critical section 1
lock.unlock();
// Non-critical work
lock.lock();
// Critical section 2
```

`std::scoped_lock` (C++17) can lock multiple mutexes simultaneously without deadlock. It uses a deadlock-avoidance algorithm internally:

```cpp
std::scoped_lock lock(mtx1, mtx2);  // Locks both, deadlock-free
```

`std::shared_mutex` (C++17) implements a reader-writer lock. Multiple readers can hold the lock simultaneously, but writers need exclusive access:

```cpp
std::shared_mutex rw_mutex;

void reader() {
    std::shared_lock lock(rw_mutex);  // Multiple readers OK
    // Read data
}

void writer() {
    std::unique_lock lock(rw_mutex);  // Exclusive access
    // Modify data
}
```

`std::condition_variable` allows threads to wait for conditions:

```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_for_ready() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });  // Handles spurious wakeups
    // Proceed when ready is true
}

void set_ready() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // Wake one waiting thread
}
```

`std::atomic` provides lock-free atomic operations for simple types:

```cpp
std::atomic<int> counter{0};
counter++;              // Atomic increment
counter.fetch_add(1);   // Same thing, explicit
```

Atomic operations use hardware support and don't require locks, making them more efficient for simple operations like counters and flags.

## 4.9 Common Interview Questions

**Q: What's the difference between mutex and semaphore?**

A mutex is for mutual exclusion — only one thread can hold it, and only the owner can release it. A semaphore is a counter — it can allow N threads to access a resource. Also, any thread can signal a semaphore, not just the one that waited. I use mutex for protecting critical sections and counting semaphores for resource pools.

**Q: Why use `std::lock_guard` instead of manual lock/unlock?**

RAII guarantees the mutex is released even if an exception is thrown. With manual lock/unlock, if code between them throws, the mutex stays locked forever. `lock_guard` locks in constructor and unlocks in destructor, so it's exception-safe and you can't forget to unlock.

**Q: When would you use a spinlock over a mutex?**

When the critical section is very short — a few instructions. The overhead of putting a thread to sleep and waking it up might exceed the time spent spinning. But only on multi-core systems — on single-core, spinning wastes the only CPU that could be running the lock holder.

**Q: Explain the condition variable spurious wakeup problem.**

A thread waiting on a condition variable can wake up even without being signaled — this is called spurious wakeup. That's why we always use a `while` loop to check the condition, not `if`. If we wake up spuriously, we check the condition, find it's still false, and go back to waiting.

**Q: How do you avoid deadlock when locking multiple mutexes?**

The classic solution is to always lock mutexes in the same global order. If everyone locks A before B, no deadlock. In C++17, `std::scoped_lock` handles this automatically — it uses a deadlock-avoidance algorithm to lock multiple mutexes safely.

## Key Takeaways

Synchronization is essential whenever multiple threads access shared mutable state. Race conditions are among the hardest bugs to debug because they're timing-dependent and non-deterministic. A program might work perfectly in development but fail in production under load.

The critical section problem has three requirements: mutual exclusion, progress, and bounded waiting. Any synchronization mechanism should be evaluated against these criteria.

Mutexes provide mutual exclusion with ownership semantics. Semaphores are more general and can control access to multiple resources or be used for signaling. Monitors encapsulate data with operations and provide automatic mutual exclusion.

In C++, always use RAII wrappers like `std::lock_guard` or `std::scoped_lock`. They're exception-safe and prevent forgetting to unlock. Use `std::atomic` for simple types like counters — it's more efficient than a mutex. Use condition variables with `while` loops, never `if` statements.

Spinlocks are appropriate only for very short critical sections on multi-core systems. For everything else, use sleeping locks that don't waste CPU cycles.
