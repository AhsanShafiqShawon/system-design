# Understanding Throughput

## What is Throughput?

Throughput is the **rate at which a system produces or processes output over a given period of time**.

It's a general concept that appears across many domains:

- **Computing/Software** — amount of work a system completes per unit of time (requests/sec, transactions/sec, bytes/sec)
- **Networking** — actual data transfer rate over a connection (Mbps, Gbps); distinct from *bandwidth* (the theoretical maximum)
- **Databases** — how many queries or writes a database can handle per second under load
- **Manufacturing/Operations** — number of units produced per hour/day (central to Goldratt's Theory of Constraints)

### Key Distinctions

| Concept | What it measures |
|---|---|
| **Throughput** | Volume of work completed over time |
| **Latency** | Speed of a single operation |
| **Bandwidth** | Theoretical capacity (ceiling) |
| **Goodput** | Application-level useful data, minus protocol overhead |

> A system can have high throughput but also high latency (e.g., batch processing).  
> Throughput is always ≤ bandwidth — bandwidth is the ceiling, throughput is what you actually achieve.

---

## Throughput in Computing & Software — A Deep Dive

### Core Definition

Throughput is the **amount of work successfully completed by a system per unit of time**. The "work" varies by context — it could be requests, transactions, bytes, jobs, or instructions.

---

## Levels of Throughput

### 1. CPU Level — Instruction Throughput

The CPU executes instructions. Throughput here is measured in **IPC (Instructions Per Cycle)** or **MIPS (Millions of Instructions Per Second)**.

Modern CPUs improve instruction throughput via:

- **Pipelining** — overlapping stages of multiple instructions (fetch → decode → execute → write-back). While one instruction is executing, the next is being decoded.
- **Superscalar execution** — multiple execution units run instructions *in parallel* within a single cycle.
- **Out-of-order execution** — the CPU reorders instructions to avoid stalls (e.g., waiting on a memory fetch).

> **Key insight:** A single instruction's latency doesn't change, but throughput increases because many instructions are in-flight simultaneously.

---

### 2. Memory Level — Memory Bandwidth

Memory throughput = **bytes transferred between RAM and CPU per second** (GB/s).

Factors affecting it:
- Bus width (64-bit, 128-bit channels)
- Clock speed of RAM (DDR4 vs DDR5)
- Number of memory channels (dual-channel doubles throughput)

The cache hierarchy exists *specifically* to protect CPU throughput from slow RAM throughput:

| Memory Type | Approximate Latency |
|---|---|
| L1 cache hit | ~4 cycles |
| RAM fetch | ~200 cycles |

A RAM fetch stall can destroy pipeline throughput — hence why caches matter so much.

---

### 3. Disk / I/O Level

Measured in **MB/s** (sequential) or **IOPS** (random reads/writes).

| Metric | Description | Typical Values |
|---|---|---|
| Sequential throughput | Reading/writing large contiguous blocks | NVMe: 3,000–7,000 MB/s; HDD: 100–200 MB/s |
| IOPS | I/O Operations Per Second (random access) | NVMe: ~1,000,000; HDD: ~100–200 |

IOPS matters more for databases doing many small random reads/writes — this is why database performance is so sensitive to storage type.

---

### 4. Network Level

Measured in **Mbps / Gbps**.

- **Bandwidth** = pipe capacity (theoretical max)
- **Throughput** = actual data successfully delivered (always lower due to overhead, retransmits, congestion)
- **Goodput** = application-level useful data, stripping away protocol headers and retransmissions

> A 1 Gbps link might deliver only 700 Mbps of goodput under real conditions.

---

### 5. Application / Service Level

This is where most software engineers operate day-to-day.

Measured in:
- **RPS** — Requests Per Second
- **TPS** — Transactions Per Second
- **QPS** — Queries Per Second (databases)

#### What Limits Application Throughput?

Every application has a **bottleneck** — one resource that constrains the whole system. Common ones:

| Bottleneck | Symptom |
|---|---|
| CPU-bound | CPU at 100%, throughput stops scaling with more threads |
| I/O-bound | Threads blocked waiting on disk/network |
| Memory-bound | GC pressure, high memory bandwidth usage |
| Lock contention | Threads stalled waiting on synchronized blocks |
| DB bottleneck | Slow queries, connection pool exhausted |
| Network saturation | Packet loss, retransmits, high RTT |

---

## Throughput vs. Latency — The Deep Relationship

This is the most important and most misunderstood dynamic.

### Little's Law

Fundamental law of queueing theory:

```
L = λ × W
```

Where:
- **L** = average number of requests in the system (concurrency)
- **λ** (lambda) = throughput (requests/sec)
- **W** = average latency (seconds)

Rearranged: **`Throughput = Concurrency / Latency`**

**Implication:** To increase throughput, you either:
1. Increase concurrency (more parallel workers), or
2. Reduce latency (make each request faster)

### The Throughput/Load Curve

At low load, throughput and latency are independent. As you approach system capacity, queues form, latency rises, and throughput plateaus — then *collapses* (the **Universal Scalability Law** effect):

```
Throughput
    |          ____--------
    |      ___/            \   ← collapse under overload
    |   __/
    |  /
    | /
    |/______________________ Load (concurrency)
```

This is why **load testing matters** — you need to find where your knee curve is before production traffic does.

---

## How Throughput is Increased in Practice

### Concurrency & Parallelism
- Thread pools, async I/O, non-blocking event loops (Node.js, Netty, WebFlux)
- Reactive/async programming avoids threads sitting idle during I/O waits

### Batching
- Instead of processing 1 item per operation, process N
- Examples: JDBC batch inserts, Kafka consumer batch polling, bulk Elasticsearch indexing
- Trades **latency** for **throughput**

### Pipelining
- Send the next request before the response of the previous one arrives
- Examples: HTTP/2 multiplexing, Redis pipelining, JDBC pipelined queries

### Caching
- Serve results from memory instead of recomputing or re-fetching
- Dramatically increases throughput for read-heavy workloads

### Connection Pooling
- Reuse expensive connections (DB, HTTP) instead of creating new ones per request
- HikariCP for JDBC — a major throughput lever in Java apps

### Horizontal Scaling
- Add more instances behind a load balancer
- Throughput scales roughly linearly — until a shared resource becomes the bottleneck (usually the DB)

### Reducing Lock Contention
- Locks serialize execution and destroy concurrency
- Use lock-free structures, optimistic locking (MVCC in DBs), partitioned data structures

---

## Throughput in the JVM

### GC and Throughput

Every GC pause stops application threads ("stop-the-world"). GC tuning is fundamentally a **throughput vs. latency tradeoff**:

| GC Algorithm | Optimizes For |
|---|---|
| Parallel GC | Raw throughput (accepts longer pauses) |
| G1GC | Balanced (low pause + decent throughput) |
| ZGC / Shenandoah | Low latency (sub-millisecond pauses) |

> High allocation rate → frequent GC → throughput degradation. Reducing object churn is often the most effective JVM throughput optimization.

### Thread Pool Sizing

- **CPU-bound** work: `threads ≈ CPU cores`
- **I/O-bound** work: `threads = cores / (1 - blocking_coefficient)` *(Brian Goetz formula)*

Too few threads → CPU idle during I/O waits → low throughput.  
Too many threads → context switching overhead → throughput degrades.

### HikariCP (DB Connection Pool)

HikariCP's own research shows that **a small pool + a short queue beats a large pool** for throughput, because large pools cause DB-side lock contention and context switching.

---

## Measuring Throughput

| Tool | Level |
|---|---|
| **JMH** | Java Microbenchmark Harness — method/component level |
| **Gatling / k6 / JMeter** | HTTP-level load testing |
| **async-profiler** | Flame graphs to find where CPU time goes |
| **Micrometer + Prometheus + Grafana** | Production RPS/TPS dashboards |

> **Key principle:** Always measure under realistic concurrency, not just in isolation. A method that takes 1ms alone might degrade to 50ms under 500 concurrent threads due to contention.

---

## Summary

Throughput is not just "how fast" — it's **how much work your system sustains over time under real load**, shaped by:

- **Concurrency** — how many things run in parallel
- **Bottlenecks** — the one resource that caps everything else
- **Queueing behavior** — how the system behaves as load approaches capacity
- **The latency tradeoff** — the fundamental tension between speed-per-request and volume-per-second