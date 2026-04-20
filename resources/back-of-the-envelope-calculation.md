# Back-of-the-Envelope Calculations

> A framework for estimating system performance and choosing the right design — before writing a single line of code.

---

## What Is This?

When designing a system, you often face multiple approaches and need to pick the best one. Building and testing every option is too expensive. The solution: **back-of-the-envelope calculations** — quick mental math estimates that let you compare designs upfront.

This technique was championed by **Jeff Dean**, one of Google's most influential engineers (BigTable, MapReduce, search infrastructure), and it's considered a core skill for any serious software engineer.

> *"An important skill for every software engineer is the ability to estimate the performance of alternative systems using back-of-the-envelope calculations, without having to build them."*
> — Jeff Dean

---

## The Latency Cheat Sheet

To estimate anything, you need to know roughly how long operations take. Memorize these:

| Operation | Latency | Human-Friendly |
|---|---|---|
| L1 cache reference | 0.5 ns | — |
| Branch mispredict | 5 ns | 10× L1 |
| L2 cache reference | 7 ns | 14× L1 |
| Mutex lock/unlock | 100 ns | — |
| Main memory reference | 100 ns | 200× L1 |
| Compress 1K bytes (Zippy) | 10,000 ns | 10 µs |
| Send 2K bytes over 1 Gbps network | 20,000 ns | 20 µs |
| Read 1 MB sequentially from memory | 250,000 ns | 0.25 ms |
| Round trip within same datacenter | 500,000 ns | 0.5 ms |
| Disk seek | 10,000,000 ns | 10 ms |
| Read 1 MB sequentially from network | 10,000,000 ns | 10 ms |
| Read 1 MB sequentially from disk | 30,000,000 ns | 30 ms |
| Send packet CA → Netherlands → CA | 150,000,000 ns | 150 ms |

### Key Intuitions from These Numbers

- **Memory is fast; disk is ~120× slower** for sequential reads (0.25ms vs 30ms for 1MB).
- **Disk seeks are brutal** — 10ms each. Minimize them at all costs.
- **Network within a datacenter is fast** (~0.5ms RTT), but cross-continent is ~300× slower.
- **Writes are ~40× more expensive than reads.** Global shared mutable state serializes transactions and kills throughput.
- **Compression is cheap.** Compressing 1K takes only 10µs but can halve your network payload.

---

## The Core Example: 30 Image Thumbnails

Imagine you're building an image search results page that loads **30 thumbnails**, each **256KB**, from disk (disk read speed ≈ 30 MB/s).

### Design 1 — Serial (one at a time)

Read each image one after another: seek → read → seek → read → ...

```
Time = (number of seeks × seek time) + (total data / disk throughput)
     = 30 × 10ms + (30 × 256KB) / 30MB/s
     = 300ms + (7,680KB / 30,000KB/s)
     = 300ms + 256ms
     ≈ 556ms
```

**Result: ~560ms** — over half a second. Not great.

---

### Design 2 — Parallel (all at once)

Issue all 30 reads simultaneously. The seeks happen in parallel, so you only pay for **one seek** and **one read** in terms of wall-clock time.

```
Time = 1 seek + (1 image read)
     = 10ms + (256KB / 30MB/s)
     = 10ms + 8ms
     = 18ms
```

Add real-world variance from competing disk activity and you get roughly **30–60ms**.

**Result: ~18–60ms** — 10–30× faster than serial.

---

### The Takeaway

You didn't need to build either system. The math told you parallel wins — decisively.

---

## Extending the Analysis: Follow-Up Questions

Once you have rough numbers, you can keep asking questions:

### Should I cache thumbnails?

A cache write might take ~1ms (e.g., writing to Redis). If the same image is requested frequently, you pay 1ms once and avoid 10ms disk seeks on every subsequent hit. For a hot image hit 100 times a day, caching saves ~(100 × 10ms) − 1ms = **~999ms per day**.

Worth it? Yes, if your hit rate is high. No, if images are rarely re-requested (e.g., long-tail search queries).

### Should I cache the entire 30-image set as one entry?

- **Pro:** One cache lookup vs. 30. Simpler.
- **Con:** A single change (e.g., one thumbnail updated) invalidates the whole set.
- **Rule of thumb:** Cache the whole page if the set is stable; cache individually if images update independently.

### Should I precompute thumbnails?

Generating a thumbnail from a raw image at request time might take ~50–200ms (CPU-bound resize). If you serve 1,000 requests/second, that's up to **200 seconds of CPU time per second** — clearly unsustainable. Precomputing and storing thumbnails trades storage (cheap) for CPU at request time (expensive). Almost always worth it at scale.

---

## Another Example: Database Read vs. Cache

You have a user profile service. Fetching a profile requires:

- **From DB (disk-backed):** ~1 disk seek + network = 10ms + 0.5ms ≈ **10–15ms**
- **From in-memory cache (Redis):** ~0.25ms memory read + 0.5ms network ≈ **0.5–1ms**

At **10,000 requests/second**:

| Approach | Total latency budget/sec |
|---|---|
| DB reads | 10,000 × 10ms = **100,000ms = 100s of DB time** |
| Cache hits | 10,000 × 0.5ms = **5,000ms = 5s of cache time** |

The cache is **20× more efficient**. Even a 90% cache hit rate turns 100s of DB time into: `10% × 100s + 90% × 5s = 10s + 4.5s = 14.5s` — still a **7× improvement** with a simple cache layer.

---

## Another Example: Synchronous vs. Async Writes

You're logging every user action to a database. Each write takes ~5ms (disk seek + commit).

At **500 writes/second** (synchronous):

```
500 × 5ms = 2,500ms = 2.5s of write time per second
```

You're already over capacity on a single disk. Options:

1. **Batch writes** — collect 100 writes, flush every 100ms → 5 batched seeks instead of 500 individual ones → **100× fewer disk operations**.
2. **Async/queue** — write to an in-memory queue (0.1µs), flush asynchronously → user sees sub-millisecond response; disk catches up in the background.
3. **Append-only log** — sequential writes avoid seeks entirely. Reading 1MB sequentially from disk takes 30ms; appending is similarly fast. This is why Kafka, write-ahead logs, and LSM-trees exist.

---

## The Mental Framework

When evaluating any design decision, run through this checklist:

1. **Identify the bottleneck.** Is it CPU? Memory? Disk I/O? Network?
2. **Estimate the cost of each operation** using the latency table.
3. **Multiply by frequency.** One disk seek is fine; 10,000/second is not.
4. **Compare alternatives** mathematically before building.
5. **Know your own system's numbers.** Generic estimates get you started; profiling your actual subsystems gets you accuracy.

---

## Lessons to Internalize

- **Disk seeks are your enemy.** Design to minimize them — batch, cache, or go sequential.
- **Memory is your friend.** An in-memory operation is often 100–1000× faster than a disk or network operation.
- **Writes are expensive.** Architect to reduce write contention. Make writes parallel where possible.
- **Network within a DC is fast; cross-region is slow.** Don't assume a round trip is free.
- **Compression is almost always worth it** on network-bound data. The CPU cost is trivial compared to bandwidth savings.
- **Monitor everything.** You can only make good estimates if you have real data from production. Guess first, measure to validate.

---

## Quick Reference Card

```
L1 cache:        0.5 ns
L2 cache:        7 ns
RAM access:      100 ns
SSD read:        ~100 µs   (not in original list, but useful)
Disk seek:       10 ms     ← this is the big one
DC round trip:   0.5 ms
Cross-continent: 150 ms

Memory read (1MB):   0.25 ms
Network read (1MB):  10 ms
Disk read (1MB):     30 ms   ← 120× slower than memory
```

**Rules of thumb:**
- Disk is ~120× slower than memory for sequential reads
- Cross-continent network is ~300× slower than in-datacenter
- Writes are ~40× more expensive than reads
- Compression cost (1K): ~10µs — almost always free
```