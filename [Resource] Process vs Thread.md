# Process vs Thread

> A deep dive from OS fundamentals to JVM internals — and then in plain English.

---

## Table of Contents

1. [The Basics](#1-the-basics)
2. [Deep Dive: OS Fundamentals to JVM Internals](#2-deep-dive-os-fundamentals-to-jvm-internals)
   - [What the OS Actually Manages](#21-what-the-os-actually-manages)
   - [Context Switching](#22-context-switching)
   - [Memory: The Deep Picture](#23-memory-the-deep-picture)
   - [Synchronization Primitives](#24-synchronization-primitives)
   - [The JVM Threading Model](#25-the-jvm-threading-model)
   - [The Scheduler](#26-the-scheduler)
   - [What Happens When You Start a Java Program](#27-what-happens-when-you-start-a-java-program)
3. [Plain English Explanation](#3-plain-english-explanation)

---

## 1. The Basics

### Process

A **process** is an independent program in execution. It has its own memory space (heap, stack, code, data segments), its own file descriptors, and its own OS resources.

- Isolated: one process cannot directly access another's memory
- Heavyweight to create (OS must allocate a new address space)
- Communication between processes requires IPC (pipes, sockets, shared memory)
- A crash in one process doesn't affect others

### Thread

A **thread** is a unit of execution *within* a process. Multiple threads share the same memory space (heap, globals, open files) but each has its own stack and program counter.

- Lightweight to create (no new address space needed)
- Shared memory makes communication fast but introduces race conditions
- A crashed thread (e.g. unhandled exception) can bring down the whole process
- Requires synchronization primitives (locks, semaphores) to avoid data corruption

### Side-by-side Comparison

| Aspect | Process | Thread |
|---|---|---|
| Memory | Own address space | Shared with parent process |
| Creation cost | High | Low |
| Communication | IPC (slow, complex) | Shared memory (fast, risky) |
| Isolation | Strong | Weak |
| Failure impact | Isolated | Can crash whole process |
| Context switch | Expensive | Cheaper |

---

## 2. Deep Dive: OS Fundamentals to JVM Internals

### 2.1 What the OS Actually Manages

The OS kernel is the source of truth for both processes and threads.

#### Process = Protection Boundary

When a process is created, the OS sets up a **virtual address space** — a private illusion that the process owns all of memory. The hardware MMU (Memory Management Unit) enforces this. If process A tries to read process B's memory, the MMU raises a fault → segfault/access violation.

The virtual address space is divided into regions:

```
High addresses
┌─────────────────┐
│   Kernel space  │  ← OS only, user code cannot touch
├─────────────────┤
│   Stack         │  ← grows downward, per-thread
│       ↓         │
│   (gap)         │
│       ↑         │
│   Heap          │  ← grows upward, malloc/new
├─────────────────┤
│   BSS segment   │  ← uninitialized globals
│   Data segment  │  ← initialized globals
│   Text segment  │  ← executable code (read-only)
Low addresses
```

#### Thread = Execution Context Within a Process

A thread is the OS scheduler's unit of work. It tracks:

- **Program Counter (PC)** — which instruction executes next
- **Register set** — CPU registers (rax, rbx, rsp, rbp, etc.)
- **Stack pointer** — points to its own private stack region
- **Thread-local storage (TLS)** — small per-thread data area

Everything else — heap, code, file descriptors, signal handlers — is shared with all threads in the process.

---

### 2.2 Context Switching

This is where the performance difference becomes concrete.

#### Thread Context Switch

The OS saves the current thread's registers + stack pointer into a **Thread Control Block (TCB)**, loads another thread's TCB, and resumes. Since the address space doesn't change (same process), the MMU page tables stay the same.

**Cost:** ~1–10 microseconds. Main overhead is register save/restore + potential cache/TLB warming.

#### Process Context Switch

Same as above, **plus**: the OS must swap the entire **page table** (virtual→physical memory mapping). This invalidates the TLB (Translation Lookaside Buffer) — the hardware cache for address translations.

**Cost:** ~50–100+ microseconds. The TLB flush is brutal — subsequent memory accesses all miss until the TLB rewarms. This is the *real* cost, not the switch itself.

---

### 2.3 Memory: The Deep Picture

#### Stack Memory

Each thread gets a fixed-size stack (default ~512KB–8MB depending on OS/JVM flags). The OS often uses **guard pages** — unmapped pages at the bottom of the stack. If a thread overflows into a guard page, the MMU raises a fault → `StackOverflowError` in Java.

Stack memory is **fast** because:
- Allocation is just decrementing the stack pointer (a single `SUB rsp, N` instruction)
- It's LIFO — excellent CPU cache locality
- No GC involvement

#### Heap Memory

Shared across all threads. Allocation requires coordination (hence why `malloc` uses locks internally). In the JVM, the GC manages this space entirely, divided into regions (Eden, Survivor, Old Gen, etc.).

**Critical implication:** any object on the heap is *reachable* by any thread that holds a reference to it. This is the root cause of all thread-safety problems.

#### False Sharing (subtle, important)

CPUs operate on **cache lines** (~64 bytes). If two threads write to *different* variables that happen to sit on the *same cache line*, the cores fight over that cache line — a condition called **false sharing**. Performance can degrade dramatically despite no logical data sharing. Java's `@Contended` annotation (and JVM padding) exists to combat this.

---

### 2.4 Synchronization Primitives

Shared memory is dangerous. The OS and hardware provide tools:

#### Mutex (Mutual Exclusion Lock)

Only one thread holds it at a time. Others **block** (the OS suspends them). In Java: `synchronized` block or `ReentrantLock`.

Key property: **mutual exclusion + memory visibility**. Acquiring a lock flushes the CPU's store buffer → ensures you see writes from threads that previously held the lock.

#### Volatile (Java)

Not a lock, but guarantees **visibility**. A write to a `volatile` field is immediately flushed to main memory; a read always fetches from main memory. Prevents CPU/compiler reordering around that variable.

Weaker than a lock — no atomicity for compound operations (check-then-act).

#### The Java Memory Model (JMM)

Java's spec defines **happens-before** relationships. If action A happens-before B, then B sees all of A's writes. Established by:

- Thread start/join
- Lock release/acquire
- Volatile write/read
- `final` field writes in constructors

Without a happens-before edge, the JVM and CPU are free to reorder your code. This is why unsynchronized access to shared state produces undefined behavior even on seemingly simple reads.

---

### 2.5 The JVM Threading Model

#### Platform Threads (Java 1.0–20)

`new Thread()` → JVM calls `pthread_create` (Linux) / `CreateThread` (Windows) → 1:1 OS thread.

Each platform thread:
- Consumes ~1MB+ of native stack (OS-side)
- Consumes a JVM stack (configurable via `-Xss`)
- Is a kernel-scheduled entity

This is why "one thread per request" web servers hit a wall around ~10K concurrent threads — you run out of memory and scheduler capacity.

#### Virtual Threads (Java 21+, JEP 444)

A **virtual thread** is a JVM-managed thread multiplexed onto a pool of **carrier threads** (platform threads).

```
Virtual Thread 1 ──┐
Virtual Thread 2 ──┼──► Carrier Thread (OS Thread) 1
Virtual Thread 3 ──┘
Virtual Thread 4 ──┐
Virtual Thread 5 ──┼──► Carrier Thread (OS Thread) 2
Virtual Thread 6 ──┘
```

When a virtual thread **blocks** (e.g. waiting on I/O), the JVM **unmounts** it from the carrier thread — saves its stack to the heap — and mounts a different virtual thread. The OS thread never blocks.

Implications:
- Stack is stored **on the heap** (not native stack) — starts tiny, grows as needed
- You can have millions of virtual threads
- CPU-bound tasks get no benefit — blocking is what they optimize
- `synchronized` blocks can **pin** a virtual thread to its carrier (a known limitation, being addressed)

#### JVM Stack vs Native Stack

Every thread (platform or carrier) has both:
- A **native stack** — used by JVM internals and JNI
- A **JVM stack** — made of **stack frames**, one per method call

Each stack frame contains:
- Local variable array (slot 0 = `this` for instance methods)
- Operand stack (where bytecode operations happen)
- Reference to the runtime constant pool
- Return address

When you call a method, a new frame is pushed. When it returns, the frame is popped and the return value is placed on the caller's operand stack.

---

### 2.6 The Scheduler

The OS scheduler decides which thread runs on which CPU core at any given moment.

#### Preemptive Scheduling

The OS can forcibly **preempt** (pause) a running thread mid-instruction via a timer interrupt, even if the thread doesn't cooperate. This is why you can have a race condition between `i++` (which compiles to read → increment → write — three separate steps) — the thread can be preempted between any of them.

#### Scheduling States

```
NEW → RUNNABLE ⇌ RUNNING → TERMINATED
                    ↕
                 BLOCKED/WAITING
```

A thread moves to BLOCKED when it tries to acquire a held lock, waits on I/O, or calls `Thread.sleep()`. The OS removes it from the run queue — it consumes no CPU. When the condition resolves, it moves back to RUNNABLE.

---

### 2.7 What Happens When You Start a Java Program

```bash
java -Xmx512m com.miniagoda.Main
```

1. OS creates a new **process** with its own virtual address space
2. OS loader maps the JVM binary (`libjvm.so`) into that address space
3. JVM initializes: sets up heap regions, metaspace, internal structures
4. JVM creates the **main thread** (a platform thread → OS thread)
5. The main thread's JVM stack gets its first frame: `main(String[] args)`
6. `ClassLoader` hierarchy is bootstrapped (Bootstrap → Platform → App)
7. Your `Main` class is located, loaded, linked, initialized
8. `main()` bytecode executes on the main thread's operand stack
9. When you create a `new Thread(() -> ...)` and call `.start()`, JVM calls `pthread_create` → new OS thread → new JVM stack
10. Both threads now run concurrently, sharing the heap

Every Spring bean, every entity object, every cached value — all of it lives on that shared heap, accessible by any thread handling any HTTP request. The framework (Spring's request thread pool) handles the concurrency, but your code must be correct under concurrent access.

---

## 3. Plain English Explanation

### The Big Idea

Your computer runs many programs at once. To do that safely and efficiently, it needs two concepts: **processes** (for isolation) and **threads** (for doing multiple things within one program).

### Process: Your Program's Private World

When you launch a program, the OS gives it a **private bubble of memory**. That bubble is an illusion — the program thinks it owns all of memory, but the hardware is quietly translating its addresses to real physical RAM behind the scenes.

The key point: **no other program can peek into that bubble**. If it tries, the hardware immediately raises an alarm and the OS kills the offending program. That's what a segfault is — you touched memory that wasn't yours.

This isolation is the whole point of a process. It's a safety boundary.

### Thread: A Worker Inside That Bubble

A thread is just a **worker running inside the process's bubble**. You can have many workers, and they all share the same room (memory). They can all read and write the same data.

Each worker has its own **notepad** (the stack) — a private scratch space for tracking what it's currently doing: which function it called, what the local variables are, where to return when done.

But the **warehouse floor** (the heap) is shared. Any worker can walk up to any shelf and grab or modify anything.

### Why Switching Between Processes is Slow

Imagine two people each working in separate locked rooms. If you need to switch from helping Person A to Person B, you have to:

- Lock A's room
- Walk to B's room
- Unlock it
- Re-learn where everything is in B's room

That "re-learning where everything is" is the expensive part. The CPU has a small memory cheat-sheet (the TLB) that helps it quickly find things in memory. When you switch processes, you have to **throw that cheat-sheet away** and rebuild it from scratch. That's slow.

Switching between threads in the *same* process is much cheaper — same room, cheat-sheet stays valid.

### Why Shared Memory is Dangerous

Here's the problem with threads sharing the warehouse floor.

Say two workers both need to update a counter. Worker A reads it (value: 5), Worker B reads it (value: 5). Worker A adds 1 and writes 6. Worker B adds 1 and also writes 6. The counter should be 7, but it's 6. One update got lost.

This is a **race condition** — the result depends on who ran first, and you can't control that.

The OS scheduler can pause a thread *at any moment* — mid-operation, between two instructions — and run a different thread. Your code isn't as atomic as it looks.

**Locks** solve this by saying "only one worker can be in this section at a time." Everyone else waits at the door.

**Volatile** is a weaker tool — it just says "always read/write this variable straight to main memory, don't cache it." It prevents stale reads but doesn't prevent race conditions on compound operations.

### The JVM's Threading World

Java threads map directly to OS threads — one Java thread equals one real OS thread. That's fine until you need *thousands* of them. Each thread eats ~1MB of memory just for its stack, and the OS gets bogged down managing them all. A server handling 10,000 simultaneous requests would need 10,000 threads — that's 10GB just for stacks.

**Virtual threads** (Java 21) solve this. Think of it like a call center with 10 phone lines but 10,000 customers. Most customers are on hold (waiting for a database, waiting for a network response). A virtual thread, when it hits a waiting point, **steps off the phone line** and lets someone else use it. When its wait is over, it gets back on a line.

The waiting threads are stored in memory (the heap) instead of tying up an OS thread. You can have millions of them cheaply.

### What Actually Happens When You Run `java Main`

1. The OS creates a new process — a private memory bubble
2. The JVM boots up inside that bubble, sets up the heap
3. One thread starts — the main thread — and begins running your code
4. Every object you create with `new` goes on the heap
5. When you start a new thread, it gets its own notepad (stack) but shares the same heap
6. All your threads are now running concurrently, all touching the same heap

### The One-Line Summary

> **A process is a protected room. Threads are workers inside that room. Workers are efficient because they share the room, but dangerous because they can step on each other's work.**

Everything in concurrency — locks, volatile, Java's memory model, virtual threads — is just tooling built to manage that last problem.

---

*Notes compiled from a deep-dive conversation on OS and JVM internals.*
