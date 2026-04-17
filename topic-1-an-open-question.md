# System Bottlenecks at 1 Million Requests Per Second

---

## Q: Suppose I have a system that receives 1 million requests per second. Which component will be broken down? CPU, RAM, or Network?

The answer is: **it depends on what each request does**, but here's a practical breakdown:

### The Three Bottlenecks

**1 million requests/second (1M RPS)** is a serious load. Let's examine each component:

---

### 🔴 Network — Most Likely the First Bottleneck

At 1M RPS, just the **packet overhead alone** is brutal:

- Each HTTP request has ~100–200 bytes of headers minimum
- 1M RPS × 200 bytes = **~1.6 Gbps just for request headers**
- Add responses, and you're easily hitting **10–40 Gbps**
- Most servers have 10GbE NICs; 40GbE/100GbE is expensive and rare

**Also**, the Linux kernel's network stack has limits on how many packets/sec it can process (typically ~1–3M PPS for a single NIC without kernel bypass techniques like DPDK).

---

### 🟡 CPU — Second Most Likely

- 1M RPS means the system has **~1 microsecond per request**
- Even a 32-core machine at 3GHz gives you only ~96,000 CPU cycles per request
- TLS handshakes, JSON parsing, business logic, and syscalls eat those cycles fast
- **Context switching** and **interrupt handling** alone can saturate CPUs

---

### 🟢 RAM — Least Likely to Break First

- RAM bandwidth (~50–100 GB/s) and capacity are rarely the primary bottleneck at this RPS
- **Unless** you're doing large in-memory lookups, caching huge datasets, or have memory leaks
- More often RAM causes issues indirectly via **cache misses** (L1/L2/L3) slowing down CPU

---

### The Real-World Answer

| Scenario | Bottleneck |
|---|---|
| Tiny requests, no TLS | **CPU** (syscall/interrupt overhead) |
| Large payloads | **Network** bandwidth |
| High connection rate (new conns) | **CPU** (TLS, TCP handshake) |
| Persistent connections (HTTP/2) | **CPU** (processing logic) |
| Stateful lookups | **RAM** / disk I/O |

### How Real Systems Handle This

To actually sustain 1M RPS on a single machine, you'd need:
- **Kernel bypass networking** (DPDK, io_uring, or RDMA)
- **Event-driven architecture** (not one thread per request)
- **100GbE NICs**
- **CPU pinning** and NUMA-aware memory allocation

Otherwise, the typical failure order is: **Network → CPU → RAM**.

Most real-world deployments at this scale use **horizontal sharding** across many machines rather than pushing a single node to its limits.

---

## Q: What does this line mean? "Most servers have 10GbE NICs"

**NIC** = **Network Interface Card** — the hardware component that physically connects a server to a network.

**GbE** = **Gigabit Ethernet** — a standard for network speed.

So **10GbE NIC** means a Network Interface Card capable of transferring data at **10 gigabits per second (Gbps)**.

### Breaking it down further:

| Term | Meaning |
|---|---|
| **NIC** | The hardware card (or chip) that handles sending/receiving network packets |
| **1GbE** | 1 Gbps — common in home/office networks |
| **10GbE** | 10 Gbps — standard in most data center servers |
| **40GbE / 100GbE** | Much faster, but expensive — used in high-performance systems |

### Why it matters for 1M RPS:

If your requests + responses average even **10KB each**:

> 1,000,000 × 10,000 bytes = **10 GB/s = 80 Gbps**

A 10GbE NIC maxes out at **1.25 GB/s**, so it would be completely saturated — that's the bottleneck.

Think of the NIC like a **pipe**: no matter how fast your CPU or RAM is, data can only flow as fast as the pipe allows.

---

## Q: Can you explain in easy language? "the Linux kernel's network stack has limits on how many packets/sec it can process"

### Think of it like a Post Office

- **Your application** is a warehouse waiting to receive packages
- **Each network packet** is a package arriving at the post office
- **The Linux kernel's network stack** is the **post office staff** that processes each package before it reaches the warehouse

### What the "network stack" actually does per packet:

Every single packet that arrives, the kernel must:

1. **Notice it arrived** (hardware interrupt — "hey, a packet is here!")
2. **Copy it** from the NIC into memory
3. **Inspect it** — is it TCP? UDP? Is it valid?
4. **Find who it belongs to** — which application/port wants this?
5. **Hand it over** to the right application

This happens **for every single packet**, thousands or millions of times per second.

### The problem at 1M RPS:

At high speed, the kernel staff gets **overwhelmed**:

- Too many interrupts → CPU spends all its time just *noticing* packets, not processing them
- This is called **interrupt storm**
- The "post office" backlog builds up faster than it can be cleared
- Packets start getting **dropped**

### The typical limit:

A standard Linux server can handle roughly **1–3 million packets/sec** before this becomes a problem — and that's *just for delivery*, before your app does anything useful with the data.

### The fix (in simple terms):

Technologies like **DPDK** essentially let your application **bypass the post office entirely** and go pick up packages directly from the loading dock — much faster, but more complex to set up.
