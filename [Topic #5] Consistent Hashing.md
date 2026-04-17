# Consistent Hashing

A complete guide from first principles to production — covering the core idea, deep technical internals, and plain-English intuition.

---

## What Is Consistent Hashing?

Consistent hashing is a technique for distributing data across nodes (servers, caches, etc.) such that **adding or removing nodes causes minimal redistribution of data**.

### The Problem It Solves

With naive hashing (`key % N` nodes), changing N means almost every key remaps to a different node — catastrophic for distributed caches or databases.

### How It Works

Imagine a circular ring numbered 0 to 2³²−1:

1. **Nodes are placed on the ring** by hashing their identifier (e.g. server IP) to a position.
2. **Keys are also hashed** to a position on the same ring.
3. **Each key is owned by the nearest node clockwise** from its position.

When a node is added or removed, only the keys between it and its predecessor on the ring are affected — typically just `1/N` of all keys.

### Virtual Nodes

A single node might land in an unlucky spot, creating uneven load. The fix: each physical node gets **multiple virtual nodes** (replicas) spread around the ring. This distributes load much more evenly and makes rebalancing smoother.

### Key Properties

| Property | Detail |
|---|---|
| **Minimal disruption** | Only ~K/N keys move when a node joins/leaves (K = keys, N = nodes) |
| **Monotonicity** | Keys don't move between existing nodes during topology changes |
| **Load balance** | Virtual nodes make distribution roughly uniform |
| **Fault tolerance** | Replication to the next N nodes on the ring is natural |

### Where It's Used

- **Distributed caches** — Memcached, Redis Cluster
- **Databases** — Cassandra, DynamoDB (for partitioning)
- **Load balancers** — to maintain session stickiness
- **CDNs** — for routing requests to edge nodes
- **Distributed hash tables** — Chord, Kademlia

### Quick Example

```
Ring:  0 ──── A(100) ──── B(200) ──── C(300) ──── 360

Key hashes to 150 → goes to B (next clockwise node)
Key hashes to 250 → goes to C
Key hashes to 350 → wraps around → goes to A
```

If node B is removed, only B's keys move to C. A and C's existing keys are untouched.

---

## Deep Dive

### The Math Behind the Ring

The ring is a modular integer space: `[0, 2³²)`. Every entity — server or key — is mapped into it with the same hash function.

```
position(x) = H(x) mod 2³²
```

The **lookup rule** is: find the smallest ring position ≥ key's position. If none exists (key is "past" the last node), wrap around to the first node. This is equivalent to a successor query on a sorted set.

```java
// Internally this is a sorted structure
TreeMap<Long, String> ring = new TreeMap<>();

// Lookup:
Map.Entry<Long, String> entry = ring.ceilingEntry(keyHash);
if (entry == null) entry = ring.firstEntry();  // wrap-around
return entry.getValue();
```

Time complexity: `O(log N)` per lookup where N is total virtual nodes.

---

### Why Naive Modulo Fails

With `node = hash(key) % N`:

- Add a node: N becomes N+1. Almost every modulo result changes. ~`(N/(N+1))` of all keys remap.
- Remove a node: same story in reverse.

For a cache, this is catastrophic — every remapped key is a cache miss, hammering your database simultaneously. This is called a **thundering herd** and it's how systems die.

Consistent hashing bounds the disruption to `K/N` keys (where K = total keys), regardless of N.

---

### Virtual Nodes: The Rebalancing Trick

Without virtual nodes, physical placement is a single roll of the dice. You might get:

```
Ring:  ──── A(5%) ──── B(8%) ──── C(87%) ────
```

Node C owns 87% of the keyspace. Virtual nodes fix this by giving each physical server multiple ring positions:

```
A → A₁(12%), A₂(38%), A₃(71%)
B → B₁(5%),  B₂(44%), B₃(90%)
C → C₁(20%), C₂(55%), C₃(82%)
```

Each physical server now owns a patchwork of arcs instead of one big slice. With enough virtual nodes, load distributes roughly uniformly by the law of large numbers. The canonical number used in production is **150 virtual nodes per physical server** (used by Cassandra by default).

---

### Adding and Removing Nodes

This is where the elegance lives. Say node B joins at position `p`:

1. Find B's predecessor on the ring (the node whose arc contains `p`), call it A.
2. Scan A's keys. Any key with `hash > A's position && hash ≤ p` now belongs to B.
3. Transfer *only those* keys. Everything else is untouched.

Removal is the mirror image — B's keys are absorbed by B's successor. No other node is disturbed.

The guarantee: **at most `2K/N` keys move** during any single topology change (the factor of 2 accounts for virtual nodes potentially creating adjacent slots owned by the same physical server).

---

### Replication

Consistent hashing extends naturally to replication. Instead of just the first clockwise node owning a key, you walk `R` steps clockwise to find `R` replicas:

```
Key hashes to position P
  → Primary replica:   first node clockwise from P      (Node A)
  → Replica 2:         second node clockwise from P     (Node B)
  → Replica 3:         third node clockwise from P      (Node C)
```

This is exactly what **Cassandra** and **DynamoDB** do. When a node fails, the replication factor means the data still exists on the next node — and that node can serve reads (or temporarily accept writes, which is called **hinted handoff**).

---

### Hotspot Detection and Bounded Loads

A subtlety: consistent hashing minimizes *key movement*, but doesn't bound *load*. A popular key (say, a celebrity's profile) always routes to the same node regardless of how many requests hit it. This is a **hotspot**.

Two mitigations:

**1. Consistent hashing with bounded loads** (Google, 2017): Assign each server a capacity threshold (e.g. 1.1× the average). Once a server hits capacity, the key overflows to the next clockwise node. This adds a minor lookup overhead but guarantees no server receives more than 1.1× the fair share.

**2. Key splitting**: Detect hot keys via request frequency, then artificially shard them — `key → key#0, key#1, ... key#M` — and distribute the shards across nodes. The application aggregates on read.

---

### Real-World Implementations

| System | Details |
|---|---|
| **Cassandra** | 150 vnodes per server by default. Uses `Murmur3Hash` (faster, better distribution than MD5). Replication factor is configurable. |
| **DynamoDB** | Proprietary consistent hashing under the hood. Each partition owns a token range; ranges split automatically when they exceed ~10GB. |
| **Redis Cluster** | 16,384 hash slots (not a ring — it's slots modulo, but similar idea). Each master owns a range of slots; resharding moves slots atomically. |
| **Chord (DHT)** | The academic original (Stoica et al., 2001). Uses finger tables for `O(log N)` routing hops. Consistent hashing is its core primitive. |
| **Memcached** (libketama) | 160 virtual nodes per server. Standard client-side implementation. |
| **Nginx/HAProxy** | Consistent hashing for upstream selection — keeps sessions sticky to a backend without a session store. |

---

### The Failure Modes Nobody Talks About

**Hash function choice matters enormously.** MD5 was used early and seems random, but its distribution has subtle clustering. Murmur3 and xxHash are preferred today — faster and empirically better distribution.

**Virtual node count is a tuning dial, not a magic number.** Too few → uneven load. Too many → metadata overhead (the ring itself takes memory; 150 vnodes × 1000 servers = 150,000 entries in your routing table).

**Cascading failure on removal.** When a node dies, its keys flood onto its successor. If the successor was already near capacity, it dies too — and the cascade propagates. This is why **capacity planning must account for N-1 node scenarios**, and why bounded-load variants exist.

**Heterogeneous hardware.** If Node A has 4× the RAM of Node B, giving them the same number of virtual nodes is wasteful. You'd assign Node A proportionally more vnodes to absorb more load. Most production systems configure vnode count per machine tier.

---

### The Mental Model in One Sentence

> A key doesn't ask "which of N nodes is this?"; it asks "who is my nearest clockwise neighbor on a shared circle?" — and that question stays stable even as neighbors come and go.

---

## Plain English Summary

### The Core Problem

Imagine you have 3 servers storing cached data. You have millions of keys, and you need a fast rule for "which server owns this key?"

The obvious rule: `server = key % 3`. Divide the key's number by 3, take the remainder. Done.

Now you add a 4th server. Suddenly the rule is `key % 4`. Almost every key gets a different answer. Your entire cache is instantly wrong. Every request misses and hammers your database at once. Bad.

### The Ring Idea

Instead of assigning keys to servers directly, imagine a giant circular clock face numbered 0 to 4 billion.

- You place your servers on this clock by hashing their names. Server A lands at 12 o'clock, Server B at 4 o'clock, Server C at 8 o'clock.
- You place your keys on the same clock by hashing them.
- Each key belongs to whichever server is *next clockwise* from it.

Now add a 4th server. It lands somewhere on the clock — say between A and B. Only the keys that were between A and the new server need to move. Everything else stays put. You've gone from "almost everything moves" to "a small slice moves."

### Virtual Nodes

If you only place each server once on the clock, you might get unlucky. Server C could randomly land in a spot where it "owns" 70% of the clock face while A and B split the other 30%. That's terrible load balancing.

The fix: place each server *multiple times* on the clock under different fake names — "Server A copy 1," "Server A copy 2," etc. Now instead of one big arc, each server owns a bunch of little scattered arcs. They average out to roughly equal shares.

More copies = more even distribution. Cassandra uses 150 copies per server by default.

### Adding and Removing Servers

When a new server joins, it only steals keys from its immediate clockwise neighbor — the one whose arc it lands inside. Nobody else is affected.

When a server dies, its keys are absorbed by the next clockwise server. Again, nobody else is affected.

**Topology changes are surgical, not global.**

### Replication

Instead of storing each key on just one server, you walk clockwise and store it on the next 3 servers. Now if one server dies, two copies of the data still exist. This is how Cassandra and DynamoDB survive hardware failures without losing your data.

### The Things That Go Wrong in Practice

**Hot keys.** A wildly popular key — a celebrity's profile, a trending news story — always routes to the same server. The ring doesn't help with that; you have to detect it and manually split the key into shards.

**Bad luck with the clock.** Even with virtual nodes, if you use a crappy hash function, servers can cluster in one half of the ring. Use Murmur3 or xxHash — they spread things out more evenly.

**Cascading failure.** A server dies, its neighbor absorbs all its traffic, *that* server gets overloaded and dies, now its neighbor gets crushed. The whole cluster can collapse like dominoes. That's why smart systems track how loaded each server already is before routing a key to it.

**Unequal hardware.** If one server has 4× the RAM, it should own 4× as many virtual nodes — otherwise you're wasting capacity.

### The One-Line Intuition

Instead of asking *"which server is this key assigned to?"* you ask *"who is this key's nearest neighbor on a shared circle?"* — and neighbors barely change when the circle gains or loses a member.
