# Core Pinning

## What is core pinning?

Core pinning is the practice of binding a process or thread to a specific CPU core, so it always runs on that designated core rather than being moved around by the OS scheduler.

By default, the OS can migrate threads freely between cores for load balancing. Core pinning overrides this by setting a **CPU affinity mask** — a bitmask telling the OS which cores a thread is allowed to run on.

---

## Why it matters

**Performance benefits:**
- **Cache warmth** — data stays in the core's L1/L2 cache, reducing misses
- **Reduced context-switch overhead** — less migration = less state movement
- **Predictable latency** — critical for real-time/low-latency systems
- **NUMA awareness** — keeps threads close to their memory on multi-socket machines

**Common use cases:**
- High-frequency trading
- Real-time audio/video processing
- Database engines
- Virtual machines / containers
- Benchmarking

---

## The hardware reality

### CPU and memory hierarchy

Each core has private L1 (~32 KB, ~4 cycles) and L2 (~256 KB, ~12 cycles) caches. All cores on a socket share an L3 (~8 MB, ~40 cycles). Beyond that is local DRAM (~80 ns) and, on multi-socket machines, remote DRAM across the QPI/UPI interconnect (~160 ns).

**Latency summary:**

| Level | Typical latency |
|---|---|
| L1 cache | ~4 cycles |
| L2 cache | ~12 cycles |
| L3 cache | ~40 cycles |
| Local RAM | ~80 ns |
| Remote RAM (cross-socket) | ~160 ns |

When a thread migrates to a new core, its hot data stays on the old core. The new core starts cold — every memory access is a cache miss until the working set is refilled.

---

## How the OS scheduler works (and why it fights you)

Linux's **CFS (Completely Fair Scheduler)** runs every ~4ms. It rebalances threads across cores based purely on load — it has no concept of cache warmth. If core 0 has 3 runnable threads and core 4 has 0, CFS will migrate one regardless of the cost.

Setting a CPU affinity mask removes the thread from the load balancer's migration candidates.

```c
// Pin current thread to core 2 (Linux)
cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(2, &mask);
sched_setaffinity(0, sizeof(mask), &mask);
```

### For hard real-time isolation, affinity alone isn't enough:

- `isolcpus=2,3` in kernel boot params — removes cores from the scheduler entirely
- `SCHED_FIFO` or `SCHED_RR` — real-time scheduling classes, bypassing CFS
- `nohz_full=2,3` — disables the scheduler tick (eliminates ~4ms jitter)
- Disable IRQ affinity on isolated cores via `/proc/irq/*/smp_affinity`

---

## SMT / Hyperthreading interactions

Each physical core exposes **two logical CPUs** to the OS. Both share the same L1, L2, and execution units. Pinning to logical CPU 0 doesn't isolate you if something else runs on logical CPU 1 — cache evictions and execution port contention from the sibling still affect you.

**Fix:** Pin to sibling pairs (control both logical CPUs on a physical core), or disable SMT:

```bash
echo off > /sys/devices/system/cpu/smt/control
```

---

## NUMA-aware pinning

On multi-socket machines, pin threads to cores on the same socket as their memory.

```bash
# Pin to cores 0-3 AND bind memory to NUMA node 0
numactl --cpunodebind=0 --membind=0 ./myapp

# Inspect topology
numactl --hardware
lstopo --of ascii
```

```c
#include <numa.h>
numa_run_on_node(0);
numa_set_membind(numa_parse_nodestring("0"));
```

---

## False sharing — the hidden trap

Two threads on different cores writing to different variables that share a **64-byte cache line** will invalidate each other's cache on every write — even with perfect pinning.

```c
// BAD: counter_a and counter_b share a cache line
struct {
    uint64_t counter_a;
    uint64_t counter_b;
} shared;

// GOOD: pad to separate cache lines
struct {
    uint64_t counter_a;
    char     pad[56];
    uint64_t counter_b;
} shared __attribute__((aligned(64)));
```

Core pinning can make false sharing *worse* — you've guaranteed the threads never escape their contested cores.

---

## How to do it

| Platform | Method |
|---|---|
| Linux (shell) | `taskset -c 2 ./myapp` |
| Linux (C) | `sched_setaffinity()`, `pthread_setaffinity_np()` |
| Windows | `SetThreadAffinityMask()`, `SetProcessAffinityMask()` |
| Python | `psutil.Process().cpu_affinity([2])` |
| Java | `thread-affinity` library (OpenHFT) |

---

## Measuring it

```bash
# Check current affinity
taskset -p <pid>

# Cache miss rate
perf stat -e cache-misses,cache-references,cycles,instructions -p <pid>

# Scheduler migration events
perf stat -e sched:sched_migrate_task ./myapp

# NUMA memory access pattern
numastat -p <pid>
```

Well-pinned, NUMA-aware threads typically see cache miss rates **below 1%**. Unpinned threads on a busy multi-socket server can hit **10–20%**.

---

## The simple explanation (office analogy)

- **Cores are desks.** Each desk has a small personal drawer (L1/L2 cache) for the worker's most-used papers.
- **Without pinning**, the office manager (OS scheduler) moves workers between desks every few milliseconds for load balancing. Every move means fetching all their papers from scratch.
- **Core pinning** tells the manager: *this worker never moves.* Their papers are always at their desk.
- **NUMA** = two separate office buildings connected by a slow bridge. Keep workers and their files in the same building.
- **Hyperthreading** = two workers sharing one desk. Pinning your worker there doesn't help if a stranger is also using the drawer.
- **False sharing** = two workers at different desks whose papers are stapled together. Every edit forces the other to refetch.

> **One-sentence summary:** Core pinning stops the OS from shuffling your thread between CPU cores, so the core's fast local memory stays full of exactly the data your thread needs.

---

## Trade-offs

| Pro | Con |
|---|---|
| Predictable, low latency | Imbalanced load if pinned core gets busy |
| Cache warmth maintained | OS can no longer optimize dynamically |
| NUMA locality guaranteed | Requires careful design to avoid starvation |
| Eliminates migration overhead | Complexity increases significantly |
