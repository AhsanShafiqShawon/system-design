# System Performance Concepts: Latency, Throughput, and Everything In Between

---

## 1. Latency vs. Throughput — The Racing Car and the Truck

These two concepts are the heartbeat of any performance conversation, and they are **not the same thing**.

**Latency** is how long it takes for a single unit of work to complete — from the moment it starts to the moment it finishes. Think of it as the *speed of one trip*.

**Throughput** is how much total work a system can do in a given period of time. Think of it as *volume delivered per hour*.

### The Analogy

Imagine you need to move cargo from City A to City B.

| | Racing Car | Cargo Truck |
|---|---|---|
| **Speed (Latency)** | Blazing fast — arrives in 1 hour | Slow — arrives in 5 hours |
| **Cargo (Throughput)** | Tiny — carries 1 suitcase | Massive — carries 20 tonnes |

- A **racing car** has ultra-low latency — one package arrives almost instantly. But its throughput is terrible; it can only carry a tiny amount per trip.
- A **cargo truck** has high latency — it takes a long time for any single package to arrive. But its throughput is enormous — it moves a huge volume of goods per day.

Neither is universally better. It depends entirely on your problem.

- Sending a stock trade? You need the racing car. A 5-second delay could cost millions.
- Backing up a data warehouse overnight? You need the truck. You don't care how long any single byte takes — you care that *all* the data gets there.

### The Tension Between Them

Here's the uncomfortable truth: **latency and throughput often pull against each other**.

Batching is the classic example. If you group 1,000 requests together and process them at once, your throughput skyrockets — the system is doing work very efficiently. But every individual request now has to *wait* for the batch to fill up before it gets processed. Its latency just got worse.

High throughput systems often sacrifice latency. Low latency systems often sacrifice throughput. Understanding this trade-off is the first step to making good architectural decisions.

---

## 2. Building a Low-Latency System — What Should Throughput Be?

If your goal is low latency, the single most important rule is:

> **The system must never be saturated. Throughput capacity must always exceed demand.**

Here is why. When a system operates near or at its capacity ceiling, requests start queuing. A queued request isn't being worked on — it's waiting. That wait time adds directly to latency. Even if each individual operation is fast, a backlog destroys the user experience.

Think of a checkout line at a supermarket. A skilled cashier can process a customer in 45 seconds. But if 20 people are in line, *your* latency isn't 45 seconds — it's 20 × 45 seconds = 15 minutes. The *service time* is fine. The *waiting time* is the killer.

**Rule of thumb for low-latency systems:** design for roughly 50–70% of peak capacity under normal load. Leave headroom. The moment utilization creeps above ~70–80%, queuing effects start to dominate and latency degrades rapidly (more on this in the Saturation Curve section).

So the answer is: **in a low-latency system, throughput capacity should be generously higher than actual demand** — not because you need all that capacity right now, but because headroom prevents queuing, and queuing is latency's worst enemy.

---

## 3. Percentile Latency — p99 and p95

"Average latency" is a dangerous lie. It can look perfectly healthy while a significant portion of your users suffer.

**Percentile latency** tells you the truth.

- **p95 = 95th percentile** — 95% of requests complete *at or under* this time. 5% take longer.
- **p99 = 99th percentile** — 99% of requests complete *at or under* this time. 1% take longer.

These are often called **tail latencies** — they describe the slow tail of your latency distribution.

### Why This Matters

Suppose your system handles 1,000 requests per second. A p99 of 1 second means 10 requests per second are taking over a second. Over an hour, that's 36,000 slow requests. Those are real users with a bad experience.

But the reason percentiles are *critical* goes deeper than empathy for a few slow users. We'll see in later sections that these slow requests can cascade and amplify into system-wide problems.

### p99 vs p98 — Are They Really Different?

They can be wildly different. p98 being 10ms and p99 being 1,000ms (1 second) means the worst 1% of requests are **100× slower** than the worst 2%. This is a sign of a heavy-tailed distribution — likely caused by GC pauses, lock contention, cold caches, or resource exhaustion hitting only the unluckiest requests. The jump from p98 to p99 is a red flag worth investigating.

---

## 4. Throughput of 20, Need 100 — What To Do?

Your system processes 20 units of work per second. You need 100. That's a 5× gap. Here are the two constrained cases:

### Case 1 — You Cannot Increase Latency

If you can't make individual requests take longer, your only lever is **parallelism**. You need to process more requests *simultaneously*, not make each one faster or slower.

The options:

- **Horizontal scaling**: Add more instances of the service. If one instance handles 20 RPS, five instances handle 100 RPS — assuming work can be distributed evenly.
- **Async/non-blocking I/O**: If your system is I/O-bound (waiting on databases, network calls, disk), switching to a non-blocking model lets a single thread handle many in-flight requests concurrently without increasing per-request latency.
- **Connection pooling and pipelining**: Reduce the overhead per request so more can fit in the same time window.

The key insight: you're not changing *how long* each request takes — you're changing *how many* you're working on at the same time.

### Case 2 — You Have a 12–16 Core CPU

This is the same problem viewed through a hardware lens. You have real parallelism available — but only if your software can use it.

- **12–16 cores = 12–16 things happening simultaneously** (in theory). If your service is single-threaded or poorly threaded, you're leaving almost all of that on the floor.
- Move to a **multi-threaded or async architecture**. A thread pool sized to match core count (or slightly above it for I/O-bound work) lets the OS scheduler keep all cores busy.
- For CPU-bound work: use worker threads equal to the number of cores. 16 cores processing requests in parallel could give you close to 16× the throughput of a single-threaded baseline.
- For I/O-bound work: you can often go *higher* than core count in thread count, because threads spend most of their time waiting — not burning CPU.
- **Avoid shared mutable state**: with parallel threads, lock contention will kill your throughput gains. Prefer immutable data, thread-local state, or a message-passing model.

---

## 5. The Cart Button Problem — 100 Services, p99 = 1s, p98 = 10ms

A user clicks "Add to Cart." Behind that button, 100 independent microservices must all respond before the user sees a result.

Each service team proudly reports:
- p99 = 1,000ms (1 second)
- p98 = 10ms

So what does the user actually feel?

### The Math of Serial Calls (Worst Case)

If services are called one after another (serially):

Expected latency = sum of all individual expected latencies

But percentiles don't add that simply. The important insight is: **the overall p99 is not the sum of individual p99s**.

### The Fan-Out Reality

Even if services run in parallel, the total response time is determined by the **slowest service** — the maximum, not the sum.

This is called the **straggler problem**.

For N independent services each with probability `p` of being *slow* (above some threshold), the probability that *at least one* of them is slow is:

```
P(at least one slow) = 1 - P(all fast)
                     = 1 - (1 - p)^N
```

For our case:
- Each service has a 1% chance (p99 means 1% of requests are slow) of taking 1 second.
- N = 100 services.

```
P(at least one service is slow) = 1 - (0.99)^100
                                 = 1 - 0.366
                                 = 0.634
```

**63.4% of users will experience at least one service taking 1 second**, even though each individual service is only slow 1% of the time.

Put another way: what each team calls their "p1 bad case" becomes the *majority experience* for users when you fan out to 100 services.

### What Does the User Feel?

The user feels the latency of the **slowest service in the chain**. If even one of 100 services hits its p99 case, the whole cart action takes ~1 second. And with 100 services, that happens to roughly 2 out of every 3 users.

This is **Tail Latency Amplification** — covered in depth in Section 8.

**The lesson:** individual service SLOs are nearly meaningless in isolation. At the system level, you must target much more aggressive percentiles per service — p99.9 or p99.99 — to deliver a good p99 user experience end-to-end.

---

## 6. Queueing Theory, Little's Law, and Saturation Curves

### Queueing Theory in Plain English

When requests arrive faster than a system can process them, they queue up. Queueing theory is the mathematical study of what happens in that queue: how long requests wait, how the queue grows, and when the system breaks down.

Every queuing system has three parameters:
- **Arrival rate (λ)** — how many requests arrive per second.
- **Service rate (μ)** — how many requests the system can process per second.
- **Utilization (ρ)** — the ratio λ/μ. When ρ approaches 1 (100% utilization), things go very bad, very fast.

### Little's Law

Little's Law is one of the most elegant results in systems theory. It states:

> **L = λ × W**

Where:
- **L** = average number of requests in the system (in queue + being processed)
- **λ** = average arrival rate (requests per second)
- **W** = average time a request spends in the system (latency)

This is powerful because it's **universally true** — it doesn't assume anything about request distributions, service time distributions, or number of servers.

### Using Little's Law

**Example 1 — Estimating concurrency needed:**

Your API handles 500 requests per second (λ = 500), and each request takes 200ms on average (W = 0.2s).

```
L = 500 × 0.2 = 100
```

At any given moment, there are 100 requests in flight. Your system needs to support at least 100 concurrent operations. If your thread pool has 50 threads, you're already under-provisioned.

**Example 2 — Diagnosing a slowdown:**

Normally: λ = 1,000 RPS, W = 50ms → L = 50 concurrent requests.

During an incident: your monitoring shows L has jumped to 500 concurrent requests, but λ hasn't changed. By Little's Law: W = L / λ = 500 / 1,000 = 0.5s. Your average latency has jumped to 500ms. You found the problem.

**Example 3 — Capacity planning:**

You need to support λ = 2,000 RPS with a target latency of W = 100ms.

```
L = 2,000 × 0.1 = 200 concurrent requests
```

Your system must handle 200 simultaneous in-flight requests. Design your connection pools, thread pools, and async concurrency accordingly.

### Saturation Curves — Why 80% Utilization Is a Hard Ceiling

The relationship between utilization and latency is not linear. It's an exponential curve that goes vertical as you approach 100%.

```
Latency
  |                                          *
  |                                        *
  |                                      *
  |                                   **
  |                               ****
  |                        *******
  |          **************
  |__________________________________________
  0%      50%      70%    85%  95% 100%
                       Utilization
```

At low utilization (0–50%), latency is flat and predictable. The system has plenty of spare capacity; requests are served immediately.

Around 70–80%, latency starts climbing noticeably. Requests occasionally have to wait in a short queue.

Above 85%, latency climbs steeply. Queues build faster than they drain. Even brief arrival spikes cause latency to spike.

At 95–100%, the system is effectively broken from a user experience perspective. Queues grow without bound. Latency becomes unbounded.

**Why does this happen?**

Utilization represents the fraction of time a server is busy. When utilization = 90%, a new request arriving has a 90% chance of finding the server busy. It must queue. The queue itself has a queue in front of it, and so on. The mathematics of queueing theory (specifically the M/M/1 queue model) shows that average wait time scales as:

```
Wait time ∝ ρ / (1 - ρ)
```

At ρ = 0.5: wait ∝ 1  
At ρ = 0.8: wait ∝ 4  
At ρ = 0.9: wait ∝ 9  
At ρ = 0.95: wait ∝ 19  
At ρ = 0.99: wait ∝ 99  

**Practical rule:** For latency-sensitive systems, keep utilization below 70%. For throughput-oriented batch systems, 80–85% is acceptable. Never plan to run at 90%+.

---

## 7. Napkin Math — Estimating System Performance Quickly

Before building anything, you should be able to estimate whether your design is even in the right ballpark. This is the art of napkin math: rough calculations that take 2 minutes and can save you months of wasted engineering.

### Key Numbers to Memorize

| Operation | Approximate Latency |
|---|---|
| L1 cache access | ~1 ns |
| L2 cache access | ~4 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs (100,000 ns) |
| HDD seek | ~10 ms (10,000,000 ns) |
| Network round-trip (same DC) | ~0.5 ms |
| Network round-trip (cross-region) | ~50–150 ms |
| Database query (indexed) | ~1–10 ms |
| Database query (full scan) | ~100ms–seconds |

### Throughput Estimates

| Resource | Approximate Throughput |
|---|---|
| Single CPU core | ~1 billion simple operations/sec |
| Network (1 Gbps link) | ~125 MB/s |
| SSD sequential read | ~500 MB/s – 3 GB/s |
| PostgreSQL (simple queries) | ~10,000 QPS per instance |
| Redis | ~100,000–1,000,000 ops/sec |
| Kafka | ~1,000,000 messages/sec per broker |

### A Worked Napkin Math Example

**Question:** Can a single server handle an e-commerce site with 10,000 concurrent users, each making 2 requests per minute?

Step 1 — Arrival rate:
```
λ = 10,000 users × 2 requests/min ÷ 60 seconds = ~333 RPS
```

Step 2 — Assume each request hits the database once, taking ~5ms, plus ~10ms application logic = 15ms total.

Step 3 — Little's Law:
```
L = 333 × 0.015 = ~5 concurrent requests
```

Step 4 — Utilization check: a modern server can handle hundreds to thousands of concurrent connections. 5 concurrent requests is trivially easy. A single server is fine — for now.

Step 5 — Peak traffic: assume 10× peak (Black Friday). λ = 3,330 RPS, L = 50 concurrent requests. Still manageable on a single well-tuned server, though you'd want horizontal redundancy.

This kind of quick estimate tells you whether to use a hammer or a sledgehammer — before you write a single line of code.

---

## 8. Tail Latency Amplification and Retry Storms

### Tail Latency Amplification

We touched on this in Section 5, but let's go deeper.

**Tail latency amplification** is the phenomenon where a system's user-facing latency is much worse than any individual component's latency, because users experience the *maximum* of all component latencies, not the average.

The formula:
```
P(user sees slow response) = 1 - P(ALL services are fast)
                           = 1 - ∏(1 - p_slow_i) for all services i
```

For independent services with the same p_slow:
```
= 1 - (1 - p_slow)^N
```

**Concrete implications:**

To achieve a system-level p99 of 100ms with 10 services in the critical path, each service needs a p99 far below 100ms. The exact target depends on service dependencies and fan-out patterns, but as a rough guide:

- 10 services: each service needs roughly p99.9 to deliver system-level p99
- 100 services: each service needs roughly p99.99

This is why high-scale companies obsess over p999 (99.9th percentile) and p9999 (99.99th percentile) for their individual services, even when those numbers seem absurdly conservative in isolation.

### Hedged Requests — A Mitigation Strategy

One practical approach: if a request hasn't completed within a threshold time (say, 95th percentile latency), send a duplicate request to another instance. Use whichever response arrives first and cancel the other. This trades a small increase in load for a dramatic reduction in tail latency. Google found this could reduce 99.9th percentile latency by 2–10× at a cost of ~2% increased load.

### Retry Storms

Retries are the naive fix for slow or failed requests. "If it didn't work, try again." This is sensible for isolated, transient failures. But at scale, retries can transform a minor degradation into a catastrophic outage.

**Here's the failure mode:**

1. A downstream service gets slow — maybe a database is under load, taking 2× longer than normal.
2. Clients start timing out. Their timeout logic triggers a retry.
3. Now there are 2× as many requests hitting the already-overloaded service.
4. The service gets slower. More timeouts. More retries.
5. Now there are 4× as many requests. Then 8×. The service collapses entirely.
6. The clients are now all retrying against a dead service. The retry storm spreads upstream.

This is a **positive feedback loop** — the cure makes the disease worse.

**Why does this cascade?**

Because retries don't reduce load — they increase it. A service that's struggling at 100% utilization receives 200% load from retries. The queueing math from Section 6 kicks in: at 100% utilization, wait time is theoretically infinite. Adding more requests doesn't help; it makes it immeasurably worse.

### Preventing Retry Storms

**1. Exponential backoff with jitter**

Don't retry immediately. Wait, and add randomness to prevent synchronized retry waves.

```
wait_time = min(base × 2^attempt + random_jitter, max_wait)
```

Without jitter, all clients retry at the same intervals, creating synchronized load spikes — the thundering herd problem.

**2. Circuit breakers**

A circuit breaker is like an electrical fuse. When a service's error rate or latency exceeds a threshold, the circuit "trips": instead of sending more requests (and retrying), the client immediately returns an error or a cached fallback. The circuit stays open for a cooldown period, then allows a small test request through.

This prevents clients from piling up requests against a broken service, giving it time to recover.

States:
- **Closed** (normal): requests flow through.
- **Open** (degraded): requests are rejected immediately, no downstream calls made.
- **Half-open** (recovering): one probe request is allowed. If it succeeds, close the circuit. If it fails, reopen.

**3. Retry budgets**

Set a hard limit: "No more than X% of requests can be retries." Once the retry budget is exhausted, stop retrying. This caps the multiplier effect at the source.

**4. Idempotency**

Retries are only safe if the operation is idempotent — doing it twice has the same effect as doing it once. Charge a payment card twice? Very not idempotent. Read a product name twice? Fine. Design retry-able operations to be idempotent using unique request IDs that the server deduplicates.

---

## 9. How It All Connects

These aren't independent topics — they form a single coherent mental model.

```
User experience (latency felt)
        ↑
Tail Latency Amplification
(fan-out × individual p99 = system-level degradation)
        ↑
Individual service percentile latency (p95, p99, p999)
        ↑
Queueing effects (Little's Law, saturation curves)
        ↑
System utilization (throughput demand vs. capacity)
        ↑
Architecture choices (parallelism, async, horizontal scale)
```

When a system degrades:
- Utilization climbs → saturation kicks in → queues form
- Queues form → latency climbs → p99 worsens
- p99 worsens → tail latency amplification → most users feel the pain
- Users and services retry → retry storm → utilization climbs further
- The circuit breaker has one job: break this feedback loop

When you're doing napkin math on a new system, you're estimating utilization before you build. When you're setting SLOs on a microservice, you're reasoning about tail latency amplification. When you're provisioning a thread pool, you're applying Little's Law.

Get comfortable with these mental tools and you'll be able to reason about almost any system performance problem from first principles.

---

## Quick Reference

| Concept | One-Line Summary |
|---|---|
| Latency | Time for one request to complete |
| Throughput | Requests completed per unit time |
| p99 | 99% of requests complete at or under this time |
| Little's Law | L = λ × W (concurrency = rate × latency) |
| Utilization | λ / μ — keep below ~70% for latency-sensitive systems |
| Saturation curve | Latency explodes as utilization approaches 100% |
| Tail latency amplification | System p99 ≫ individual service p99 when fan-out is high |
| Retry storm | Retries under load increase load, causing further failures |
| Circuit breaker | Stops requests to a failing service to allow recovery |
| Hedged requests | Duplicate slow requests to a second instance; use the faster response |