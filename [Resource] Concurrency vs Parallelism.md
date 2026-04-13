# Concurrency vs Parallelism

> A deep dive from concept to silicon, with a plain-English translation.

---

## Table of Contents

1. [The Core Distinction](#1-the-core-distinction)
2. [Deep Dive — Layer by Layer](#2-deep-dive--layer-by-layer)
   - [The Mental Model](#21-the-mental-model)
   - [How the OS Sees It](#22-how-the-os-sees-it)
   - [How the CPU Sees It](#23-how-the-cpu-sees-it)
   - [How the JVM Sees It](#24-how-the-jvm-sees-it)
   - [Concurrency Hazards](#25-concurrency-hazards)
   - [Parallelism Models](#26-parallelism-models)
   - [Virtual Threads (Java 21+)](#27-virtual-threads-java-21)
3. [Plain-English Translation](#3-plain-english-translation)
   - [The Basic Idea](#31-the-basic-idea)
   - [The OS Section](#32-the-os-section)
   - [The CPU Section](#33-the-cpu-section)
   - [The JVM Section](#34-the-jvm-section)
   - [Concurrency Bugs](#35-concurrency-bugs)
   - [Parallelism Models (Plain English)](#36-parallelism-models-plain-english)
   - [Virtual Threads (Plain English)](#37-virtual-threads-plain-english)
4. [The One Thing to Remember](#4-the-one-thing-to-remember)

---

## 1. The Core Distinction

**Concurrency** and **parallelism** are related but distinct concepts.

- **Concurrency** is about *dealing with* multiple things at once — structuring a program so that multiple tasks can be in progress simultaneously, even if only one is executing at any given moment. The tasks interleave. It's about composition and design.

- **Parallelism** is about *doing* multiple things at once — literally executing multiple computations at the same instant, typically on multiple CPU cores.

### The Classic Analogy (Rob Pike)

> Concurrency is juggling two tasks with one pair of hands. Parallelism is juggling with two pairs of hands simultaneously.

A single-core CPU can be concurrent (context-switching between threads) but not truly parallel. A multi-core CPU can be both.

### In the JVM Context

| | Concurrency | Parallelism |
|---|---|---|
| **Mechanism** | Threads, coroutines, async/await | Multiple cores executing threads simultaneously |
| **Java tool** | `ExecutorService`, `CompletableFuture`, `synchronized` | `ForkJoinPool`, parallel streams, multi-core scheduling |
| **Problem it solves** | Responsiveness, coordination, I/O overlap | Throughput, CPU-bound computation |
| **Risk** | Race conditions, deadlocks | Same, plus cache coherence issues |

> **Key insight:** You can have concurrency without parallelism (single-core multithreading), and parallelism without concurrency (SIMD vector operations). But in practice, parallelism requires a concurrent design to exploit it.

---

## 2. Deep Dive — Layer by Layer

### 2.1 The Mental Model

A program is a sequence of instructions. The question is: **how do multiple sequences relate to each other in time?**

- **Concurrency** is a *property of the problem or design* — you have multiple logical tasks whose lifetimes overlap. They may or may not run simultaneously.
- **Parallelism** is a *property of execution* — multiple instructions are physically computed at the same instant.

This distinction matters because:
- You can write a concurrent program that never runs in parallel (single-core)
- You can have hardware parallelism that your program never exploits
- Bugs in concurrent code can appear or disappear depending on whether parallelism is present

---

### 2.2 How the OS Sees It

The OS knows nothing about your application logic. It deals in **processes** and **threads**.

#### Process
- Independent memory space (its own virtual address space)
- Heavy to create (fork, copy page tables, file descriptors)
- Communication via IPC: pipes, sockets, shared memory
- Crash isolation: one process dying doesn't kill another

#### Thread
- Shares the process's address space (heap, static fields, open files)
- Lightweight to create (just needs its own stack + register set)
- Communication via shared memory — which is what makes it dangerous
- A crash (e.g. `SIGSEGV`) in one thread can kill the entire process

#### Context Switching

The OS scheduler gives each thread a **time slice** (typically 1–10ms). When a slice expires, or a thread blocks (on I/O, a lock, a sleep), the scheduler:

1. Saves the thread's **CPU register state** (program counter, stack pointer, general-purpose registers) into its **Thread Control Block (TCB)**
2. Loads the next thread's register state from its TCB
3. Resumes execution

This is concurrency on a single core — tasks interleave, but only one runs at a time.

**Cost of context switching:**
- Register save/restore: cheap
- TLB flush (if switching processes): expensive — the CPU's address translation cache is invalidated
- Cache eviction: significant — the incoming thread's data is likely cold in L1/L2

---

### 2.3 How the CPU Sees It

#### Multiple Cores = True Parallelism

A modern CPU has N cores. Each core has its own:
- Register file
- L1 instruction cache + L1 data cache
- L2 cache (sometimes shared with a neighbor core)
- Execution units (ALU, FPU, etc.)

All cores share L3 cache and main memory (RAM). This shared memory is exactly what makes parallel programs hard.

#### The Memory Hierarchy

| Level | Latency | Size |
|---|---|---|
| Registers | ~0 cycles | — |
| L1 cache | ~4 cycles | ~32KB (per core) |
| L2 cache | ~12 cycles | ~256KB (per core) |
| L3 cache | ~40 cycles | ~8–32MB (shared) |
| RAM | ~200 cycles | GBs |
| SSD | ~100,000 cycles | — |

A cache miss that goes to RAM is ~50x slower than an L1 hit.

#### False Sharing

Two threads on different cores write to different variables that happen to live on the same **cache line** (64 bytes). The hardware cache coherence protocol (MESI) forces the cores to negotiate ownership of that line on every write, creating invisible serialization.

# False Sharing & CPU Cache Coherence

## What is a Cache Line?

Think of a **cache line (64 bytes)** as a small "box" of memory that a CPU core works with at a time.

---

## The Problem Setup

Imagine this situation:

- **Thread A** (on Core 1) writes to variable `x`
- **Thread B** (on Core 2) writes to variable `y`
- `x` and `y` just happen to live inside the **same 64-byte box** (cache line)

### What you'd expect

> "No problem — both threads can run in parallel. They're writing to *different* variables."

### What actually happens

Modern CPUs use a protocol like **MESI** to keep caches consistent. That protocol enforces:

> **Only one core can own a cache line for writing at a time.**

So even though Core 1 only wants `x` and Core 2 only wants `y` — they are **forced to fight over the same cache line**.

---

## Step-by-Step Analogy

Imagine:

- Two people → two CPU cores
- One shared notebook page → one cache line
- Each wants to write on a **different part** of the same page

What happens:

1. Core 1 gets the page → writes to `x`
2. Core 2 wants to write → must **take the page** from Core 1
3. Core 1 wants to write again → must **take it back**
4. Repeat…

The page keeps **bouncing** between them.

### Why this is bad

| Problem | Effect |
|--------|--------|
| Each "handoff" costs time | Cache invalidation + transfer overhead |
| Threads can't overlap | They become **silently serialized** |
| Code *looks* parallel | But *performs* like it's partially single-threaded |

This is called 👉 **False Sharing**.

---

## The Key Insight

> **The CPU doesn't care about variables — it cares about cache lines.**

So even if your code looks independent:

```java
int x; // thread A writes this
int y; // thread B writes this
```

If `x` and `y` sit close in memory → **same cache line** → contention.

---

## Fixing the Mental Model

### ❌ Common misconception

> "Each core has its own cache line — they should be independent."

### ✅ Reality

Both cores **can cache the same memory block**, but only one can write at a time.

---

## How Cache Coherence Actually Works

Suppose in memory:

```
Address 0x1000 → x
Address 0x1004 → y

Cache line covers: 0x1000 → 0x103F  (contains BOTH x and y)
```

### Step 1 — Both cores read

```
Core 1 loads cache line [0x1000–0x103F] → into its L1
Core 2 loads the SAME cache line       → into its L1

Core 1 L1: [x, y]  ✓
Core 2 L1: [x, y]  ✓  (identical copies)
```

### Step 2 — Core 1 writes to `x`

MESI kicks in. Core 1 must get **exclusive ownership** and broadcasts:

> "Invalidate your copy of this cache line."

```
Core 1: owns [x, y]  (modified)
Core 2: copy → INVALID ✗
```

### Step 3 — Core 2 writes to `y`

Core 2's copy is now invalid, so it must:

1. Request the cache line from Core 1
2. Core 1 gives it up

```
Core 2: owns [x, y]  (modified)
Core 1: copy → INVALID ✗
```

### 🔁 This keeps repeating

Even though Core 1 only cares about `x` and Core 2 only cares about `y`, they are forced to **exchange the entire 64-byte line** every time either of them writes.

---

## Why Cores Share the Same Cache Line

Cache coherence works at **cache line granularity**, not variable granularity.

The CPU does **not** track:
- "`x` belongs to Core 1"
- "`y` belongs to Core 2"

The CPU only sees:
- "This **64-byte block** changed."

---

## Mental Model Summary

| Concept | Correct understanding |
|--------|----------------------|
| Cache line | Smallest unit of ownership |
| MESI | Traffic cop: only one writer at a time |
| Different variables | ≠ independent if they share a cache line |
| Private caches | Each core *may* cache the same block |

---

## One-Line Takeaway

> Two cores can hold the same cache line simultaneously. When both write to it, it causes a **ping-pong** — that's **false sharing**.

---

## What's Next?

- **Visualize** memory layout and how padding fixes false sharing
- **Relate** this to Java/C++ structs and arrays (common in performance interviews)

```java
// These two fields share a cache line — writing them from different threads
// causes cache line ping-pong between cores
class Counters {
    volatile long a; // thread 1 writes this
    volatile long b; // thread 2 writes this
}
```

**Fix:** pad them to separate cache lines (Java 8+: `@Contended`).

---

### 2.4 How the JVM Sees It

#### Threads

`java.lang.Thread` maps 1:1 to an OS thread (platform thread). Creating one allocates:
- An OS thread (~1MB stack by default on Linux)
- A JVM `Thread` object on the heap
- Registration in the JVM's thread list

This is why you don't create thousands of them.

#### The Java Memory Model (JMM)

Without synchronization, the JMM gives you almost **no guarantees**. Both the compiler and the CPU can reorder instructions for performance, as long as the reordering is invisible *within a single thread* — but it can be visible across threads.

```java
// Thread 1
data = 42;        // (1)
ready = true;     // (2)

// Thread 2
if (ready)        // (3)
    use(data);    // (4)
```

Without synchronization, Thread 2 could see `ready == true` but `data == 0`. The CPU or compiler reordered `(1)` and `(2)` from Thread 2's perspective.

#### happens-before

The JMM is defined in terms of the **happens-before** relation. If action A happens-before action B, then A's effects are guaranteed visible to B.

Happens-before edges are established by:
- `volatile` reads/writes
- `synchronized` enter/exit (monitor acquire/release)
- `Thread.start()` / `Thread.join()`
- `Lock.lock()` / `Lock.unlock()`
- `CompletableFuture` completion

If there is no happens-before chain between a write and a read of the same variable, it's a **data race**.

#### `volatile`
- Guarantees visibility: writes are flushed to main memory, reads bypass cache
- Prevents reordering across the volatile access (memory fence)
- Does **not** make compound operations atomic (`i++` on a volatile is still a race)

#### `synchronized`
- Acquires a **monitor** (intrinsic lock) on an object
- Establishes happens-before on both entry and exit
- Flushes all writes to main memory on exit, re-reads on entry
- Mutual exclusion: only one thread holds the monitor at a time

#### CAS — The Foundation of Lock-Free Programming

`java.util.concurrent` is built on `VarHandle` which exposes hardware-level **compare-and-swap (CAS)**:

> "Set this memory location to new value *only if* it currently holds expected value, atomically."

This is a single CPU instruction (`CMPXCHG` on x86) that cannot be interrupted.

`AtomicInteger.incrementAndGet()` under the hood:

```java
do {
    current = get();
    next = current + 1;
} while (!compareAndSet(current, next)); // retry if someone else changed it
```

No locks. No OS involvement. Pure CPU atomics.

---

### 2.5 Concurrency Hazards

#### Race Condition
The program's correctness depends on the relative timing of threads.

```java
if (map.containsKey(k))    // check
    return map.get(k);     // act — another thread may have removed k in between
```

#### Deadlock
Thread A holds lock 1, waits for lock 2. Thread B holds lock 2, waits for lock 1. Neither can proceed.

**Prevention:** always acquire locks in a consistent global order.

#### Livelock
Threads keep changing state in response to each other but make no progress. Like two people in a hallway stepping aside for each other repeatedly.

#### Starvation
A thread is perpetually denied access to a resource (e.g., a low-priority thread never gets scheduled because high-priority threads always run).

#### Visibility Bugs
The most subtle. Your program appears to work in testing (DEBUG mode adds synchronization implicitly via logging, `-ea` flags, etc.) but fails in production — only under load, only on multi-core machines.

---

### 2.6 Parallelism Models

#### Fork/Join (Divide and Conquer)
Split a large task recursively until subtasks are small enough to compute directly. Results are merged up the call tree. Java's `ForkJoinPool` implements **work stealing** — idle threads steal tasks from busy threads' queues, keeping all cores saturated.

Used by: parallel streams, `CompletableFuture`'s default executor.

#### Data Parallelism
Apply the same operation to many data points simultaneously.

```java
list.parallelStream().map(this::transform).collect(toList());
```

Each element is processed independently — no shared mutable state, no synchronization needed. This is the ideal case.

#### Pipeline Parallelism
Stage 1 outputs feed Stage 2 inputs concurrently — like an assembly line. Reactive streams (`Project Reactor`, `RxJava`) model this.

---

### 2.7 Virtual Threads (Java 21+)

**The problem:** Platform threads map 1:1 to OS threads. OS threads are expensive. You can't have 100,000 of them.

**Virtual threads** are JVM-managed, lightweight threads that multiplex onto a small pool of carrier (platform) threads. When a virtual thread blocks (e.g., on a database call), the carrier thread is released to run another virtual thread. The JVM parks the virtual thread's stack on the heap.

| | Platform Thread | Virtual Thread |
|---|---|---|
| Stack | ~1MB (OS-managed) | ~few KB (heap) |
| Creation cost | High | Very low |
| Block behaviour | Holds OS thread | Releases carrier thread |
| Max practical count | Thousands | Millions |

For I/O-heavy applications (like a booking service hitting a DB on every request): virtual threads let you write simple blocking code and get the scalability of async — without callback hell or reactive plumbing.

> The concurrency model is the same. The JMM still applies. Race conditions are still possible. You just get far more concurrency per machine.

---

## 3. Plain-English Translation

### 3.1 The Basic Idea

Imagine you're cooking a meal. You put pasta on to boil, and while you're waiting, you chop vegetables. You're not literally doing two things at the exact same moment — you only have two hands — but you're managing two tasks whose lifetimes overlap. That's **concurrency**.

Now imagine you have a friend in the kitchen with you. You're both chopping vegetables at the same time. That's **parallelism** — work literally happening simultaneously.

Your computer works the same way. Concurrency is about juggling multiple tasks. Parallelism is about actually doing multiple things at once on multiple CPU cores.

---

### 3.2 The OS Section

Your operating system is like an air traffic controller. It has many programs wanting to run, but only so many runways (CPU cores). It gives each program a tiny slice of time — a few milliseconds — then yanks it off the CPU and gives another program a turn. This happens hundreds of times per second, so fast that everything *feels* like it's running simultaneously even on a single core.

A **process** is basically a running program with its own private memory — like its own apartment. A **thread** is a worker inside that apartment. Multiple threads share the same space, which is convenient (they can pass data easily) but dangerous (they can mess with each other's stuff).

When the OS switches between threads, it has to save everything the thread was doing — like bookmarking a page — then restore another thread's bookmark. This is called a **context switch**. It's not free. If done too much, the CPU spends more time switching than actually working.

---

### 3.3 The CPU Section

Your CPU has multiple cores — think of them as separate workers at separate desks. They all share one filing cabinet (RAM), but each has their own small notepad on their desk (cache) for things they're working with right now.

The notepad is much faster to read than the filing cabinet. The problem is: if two workers update their own notepads with different values for the same thing, their notepads get out of sync. The CPU has to resolve this, and that negotiation takes time.

**False sharing** is a sneaky version of this: two workers are updating *different* things, but those things happen to be on the same page of the notepad. The CPU still treats it as a conflict and forces a sync, even though logically there was no collision.

---

### 3.4 The JVM Section

You'd expect that if Thread 1 writes a value, Thread 2 immediately sees it. But that's not guaranteed. The CPU and the Java compiler are both allowed to reorder and cache operations for speed — as long as the reordering is invisible within a single thread. But another thread can absolutely see the chaos.

Imagine you're writing a note and putting it in a shared inbox. You might write the note, then put it in the inbox. Simple. But the system is allowed to put a blank envelope in the inbox first, then fill in the note — as an optimization. From your own perspective, the result is the same. But someone else checking the inbox might grab the blank envelope before you've filled it in.

This is why `volatile` and `synchronized` exist — they say: "do this in the obvious order, and make sure everyone can see the result immediately."

**happens-before** is just the formal way of saying: "Thread A's work is guaranteed to be visible to Thread B." Without a happens-before chain, you have a bug waiting to happen — you just might not see it until production, under heavy load, on a specific machine.

---

### 3.5 Concurrency Bugs

| Bug | Plain-English Description |
|---|---|
| **Race condition** | Two threads both try to update the same thing and the result depends on who gets there first. Like two people buying the last concert ticket simultaneously. |
| **Deadlock** | Thread A waits for Thread B. Thread B waits for Thread A. Nobody moves. Ever. Like two cars at a one-lane bridge, each waiting for the other to reverse. |
| **Livelock** | Both threads keep reacting to each other but neither makes progress. Like two people in a hallway who keep stepping aside for each other. |
| **Visibility bug** | Thread 1 updates a value. Thread 2 never sees it because it's reading from its own stale cache. Looks fine in testing, fails silently in production. |

---

### 3.6 Parallelism Models (Plain English)

**Fork/Join** is divide and conquer. Take a big task, split it in half, keep splitting until the pieces are small enough to just do directly, then combine results back up. Java hands these pieces out to idle cores automatically.

**Data parallelism** is even simpler: do the same thing to a million items, spread across all cores. It only works cleanly when each item is independent — no shared state, no coordination needed.

**Pipelines** are like an assembly line. Stage 1 processes something and hands it to Stage 2, which is already working on the previous item. Multiple stages run concurrently. Reactive frameworks like Spring WebFlux work this way.

---

### 3.7 Virtual Threads (Plain English)

The old problem: each thread in Java maps to a real OS thread. OS threads are expensive — they take memory, they're slow to create, and the OS can only juggle so many efficiently. So if you have 10,000 users hitting your app simultaneously, you can't just make 10,000 threads.

**Virtual threads** (Java 21) solve this. They're lightweight threads managed by the JVM itself. When a virtual thread has to wait — say, for a database to respond — it just steps aside and another virtual thread uses that slot. The JVM parks it on the heap (cheap) rather than holding an OS thread open (expensive).

For something like a hotel booking service, where every request involves waiting on a database or an external service, this is significant. You can write normal, straightforward blocking code — no callbacks, no reactive gymnastics — and still handle massive concurrency because the JVM quietly handles all the juggling for you.

---

## 4. The One Thing to Remember

**Concurrency is structure. Parallelism is execution.**

Good concurrent design lets the runtime exploit parallelism when hardware permits — but the correctness of the program must hold regardless of whether execution is parallel or interleaved.

In a Spring-based modular monolith, you're almost always dealing with *concurrency* — multiple HTTP request threads hitting your services simultaneously, sharing JVM heap state — with parallelism handled implicitly by the JVM scheduler across cores.
