# Latency in Software Systems
> A conversation-based deep dive — from definition to architecture.

---

## What is Latency?

Latency is the **time delay between when something is initiated and when its effect is observed**. The term spans many disciplines:

- **Networking/Computing** — Time for a data packet to travel from source to destination (measured in ms). A "ping" of 20ms means 20ms round trip to a server.
- **Software systems** — Response time of a service or API; how long a request takes from send to response.
- **Databases** — Delay between issuing a query and receiving results.
- **Hardware** — How long a CPU waits for data from RAM (memory latency).
- **Audio/Video** — Delay between input (mic) and output (speaker).

### Key Distinctions

| Term | Definition |
|---|---|
| **Latency** | Speed of a single operation |
| **Throughput** | How many operations happen per unit of time |
| **Bandwidth** | Capacity of a channel (how much data can flow) |

A system can have high throughput *and* high latency. Bandwidth governs capacity; latency governs delay.

---

## Latency in Software Systems — A Deep Dive

### 1. What Latency Actually Measures

Latency is the **elapsed wall-clock time** between two events:

- **Request sent → Response received** (client's perspective)
- **Work started → Work completed** (system's perspective)

These are not the same thing. The gap between them includes network transit time, queuing, and serialization — all of which matter.

---

### 2. The Anatomy of a Single Request

When a client makes a request, latency is not a single number — it's a **sum of stages**:

```
Client
  │
  ├─► [1] Network: client → server (propagation + transmission delay)
  │
  ├─► [2] Server: receive & parse request (deserialization, routing)
  │
  ├─► [3] Queuing: wait for a thread/worker to pick it up
  │
  ├─► [4] Application logic: business rules, transformations
  │
  ├─► [5] I/O: database queries, external API calls, cache lookups
  │
  ├─► [6] Response serialization & write to socket
  │
  └─► [7] Network: server → client
```

**Total latency = sum of all stages.** The slowest stage dominates — this is the **critical path**.

---

### 3. Sources of Latency

#### 3a. Network Latency

- **Propagation delay**: Speed of light through fiber is ~200,000 km/s. A Bangkok → New York round trip (~15,000 km) has a *theoretical minimum* of ~150ms — physics sets this floor.
- **Transmission delay**: Time to push bits onto the wire. Dependent on bandwidth and packet size.
- **Queuing in routers**: Packets wait in buffers at each hop. Under congestion, this balloons.
- **TCP handshake overhead**: Before a single byte of your HTTP request moves, TCP spends 1 RTT on SYN/SYN-ACK/ACK. TLS adds another 1–2 RTTs.

#### 3b. I/O Latency — The Biggest Culprit

This is where most application latency hides. Order of magnitude reference:

| Operation | Typical Latency |
|---|---|
| CPU register access | ~0.3 ns |
| L1 cache | ~1 ns |
| L2 cache | ~4 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs |
| HDD random read | ~10 ms |
| Local network round trip | ~0.5 ms |
| Cross-datacenter round trip | ~30–150 ms |
| Database query (simple, indexed) | ~1–5 ms |
| Database query (unindexed, full scan) | ~100ms–seconds |
| External HTTP API call | ~50–500 ms |

When a service makes 5 sequential database calls at 3ms each, that's 15ms of pure I/O wait — with the thread doing nothing the entire time.

#### 3c. Queuing Latency — Little's Law

If a server has a thread pool of 10 and 50 requests arrive simultaneously, 40 requests **wait in queue** before even starting.

**Little's Law:** `L = λW`

| Symbol | Meaning |
|---|---|
| L | Number of requests in system |
| λ | Arrival rate |
| W | Time each request spends in system |

As utilization approaches 100%, queue time approaches infinity. This is why systems get **exponentially** slower under high load, not linearly.

#### 3d. GC Pauses (JVM)

The JVM's Garbage Collector periodically stops-the-world. During a GC pause:
- All application threads halt
- A request that was 2ms from completing now waits 50–200ms
- This manifests as **latency spikes** in percentile charts

G1GC and ZGC reduce (not eliminate) this. ZGC targets sub-millisecond pauses.

#### 3e. Serialization / Deserialization

JSON parsing is not free. Converting a 100KB JSON payload with Jackson takes ~1–5ms. Multiply by request volume and it adds up. Protobuf/Avro can be 5–10x faster.

#### 3f. Connection Pool Contention

If a DB connection pool has 10 connections and 20 threads need one, 10 threads wait. In HikariCP (common in Spring Boot), the `pool.Wait` metric exposes this.

---

### 4. How Latency Compounds: The Fan-Out Problem

A service making N downstream calls compounds latency differently depending on execution strategy:

**Sequential calls:**
```
Total = call1 + call2 + call3 = 30ms + 50ms + 20ms = 100ms
```

**Parallel calls** (CompletableFuture / reactive):
```
Total = max(call1, call2, call3) = max(30, 50, 20) = 50ms
```

Parallelizing I/O is one of the highest-leverage optimizations — but introduces coordination complexity and partial failure handling.

#### Tail Latency Amplification

If each of 10 downstream calls has a 1% chance of taking 500ms, the probability that *at least one* takes 500ms is ~10%. Overall request latency is governed by the **slowest dependency**.

---

### 5. Measuring Latency Correctly

#### Use Percentiles, Not Averages

Averages hide the distribution. The right metrics:

| Metric | What It Tells You |
|---|---|
| p50 (median) | Typical user experience |
| p95 | Worst 5% of requests |
| p99 | Worst 1% — often what SLAs target |
| p999 | Worst 0.1% — matters at scale (1M req/day = 1000 bad requests) |

A service with p50=5ms and p99=2000ms is **not** a fast service.

#### Coordinated Omission

A subtle measurement trap: if requests are sent every 100ms but a request takes 500ms, the next request is delayed. Naive load testers don't account for this, understating tail latency. HDR Histogram and tools like `wrk2` correct for it.

#### Distributed Tracing

In a modular system, trace latency *across* components. Useful tools:

- **OpenTelemetry** — Standard instrumentation
- **Jaeger / Zipkin** — Trace visualization
- **Spring Boot Actuator + Micrometer** — Metrics for JVM-based stacks

These reveal exactly which segment of a request took how long.

---

### 6. Reducing Latency

#### At the I/O Level

- **Caching** — Avoid the round trip (Redis, in-memory). Cache-hit latency ~0.5ms vs. DB at 5ms.
- **Connection pooling** — Reuse DB connections (HikariCP). Opening a new TCP+TLS connection costs 50–200ms.
- **Async / non-blocking I/O** — Don't block a thread while waiting. In Java: `CompletableFuture`, virtual threads (Java 21+), or reactive (WebFlux).
- **Batching** — Replace 100 individual queries with one `WHERE id IN (...)`.
- **Read replicas** — Distribute read load to reduce contention.

#### At the Network Level

- **Co-locate services** — Keep communicating services in the same datacenter / AZ.
- **CDN** — Push static and cacheable content to edge nodes near users.
- **HTTP/2 or gRPC** — Multiplexed connections, no head-of-line blocking, lower overhead.
- **Keep-Alive connections** — Avoid TCP handshake per request.

#### At the Application Level

- **Eliminate N+1 queries** — A loop that fires a DB query per iteration is a classic latency killer. Use joins or batch loading.
- **Index your queries** — An unindexed column on a large table turns a 2ms query into 2 seconds.
- **Lazy vs. eager loading** — Load only what the current request needs.
- **Avoid unnecessary serialization** — Don't deserialize a blob just to re-serialize it.

#### At the Architecture Level

- **Async processing** — For work that doesn't need to complete before responding, push to a queue (Kafka, RabbitMQ) and respond immediately.
- **CQRS** — Separate read and write paths and optimize each independently.
- **Edge computing** — Run latency-sensitive logic close to the user.

---

### 7. Latency Budgets

Set a **latency budget** for an end-to-end request and allocate it across components:

```
Total budget: 200ms
  ├─ Network (client ↔ server):  30ms
  ├─ Auth / middleware:           10ms
  ├─ Business logic:              20ms
  ├─ Primary DB query:            50ms
  ├─ Cache lookup:                 5ms
  ├─ External API call:           70ms
  └─ Response serialization:      15ms
```

If the external API call starts taking 150ms, the budget is blown. Latency budgets make this visible and actionable.

---

## Summary

Latency in software is a **layered problem**:

- **Physics** sets the floor
- **Architecture** sets the ceiling
- **Implementation** determines where you land in between

The key habits:

1. Measure with **percentiles**, not averages
2. **Trace** across your full call chain
3. Identify your **critical path**
4. Attack the **slowest segment** first

In a Java modular monolith, the usual suspects are unoptimized DB queries, missing indexes, sequential I/O that could be parallel, and GC pauses — all measurable and fixable.