# Resilience Patterns

> *Building systems that bend without breaking.*

---

## What Are Resilience Patterns?

In distributed systems, failures are not exceptional — they are **inevitable**. Networks drop packets, services crash, databases time out, and downstream APIs slow to a crawl. Resilience Patterns are a set of well-established design strategies that help a system **withstand, recover from, and adapt to** these failures without cascading into a full outage.

Rather than pretending failures won't happen, resilience patterns embrace the reality of a hostile environment and encode defensive behaviors directly into the architecture.

---

## What Purpose Do They Serve?

Resilience patterns exist to achieve several interrelated goals:

| Goal | Description |
|------|-------------|
| **Fault Isolation** | Prevent a failure in one component from propagating to others |
| **Graceful Degradation** | Continue serving partial or reduced functionality when something is broken |
| **Self-Healing** | Automatically recover normal operation once a dependency stabilizes |
| **Latency Control** | Prevent slow dependencies from exhausting shared resources |
| **Predictable Failure** | Fail fast with clear signals instead of hanging indefinitely |

Without these patterns, a distributed system is only as reliable as its **least reliable dependency** — and with enough dependencies, that means near-constant instability.

---

## The Five Core Patterns

---

### 1. Circuit Breaker Pattern

**Analogy:** An electrical circuit breaker that trips when current exceeds a safe threshold, protecting the wiring.

#### How It Works

The Circuit Breaker wraps calls to a remote service and monitors for failures. It operates in three states:

```
  [CLOSED] ──── too many failures ────► [OPEN]
      ▲                                    │
      │                                    │ after timeout
      │                                 [HALF-OPEN]
      └──────── call succeeds ─────────────┘
```

| State | Behavior |
|-------|----------|
| **Closed** | All calls pass through normally. Failures are counted. |
| **Open** | All calls are immediately rejected without even attempting the remote call. |
| **Half-Open** | A limited number of trial calls are allowed. If they succeed, the breaker closes again; if they fail, it reopens. |

#### Purpose

- Stops hammering a struggling dependency with requests it cannot handle
- Gives the failing service time and space to recover
- Fails **fast and predictably** instead of waiting for timeouts

#### Example Use Case

Your payment gateway starts returning 500 errors. Without a circuit breaker, every user checkout hangs for 30 seconds before failing. With one, the circuit opens after 5 failures and requests fail in milliseconds — preserving threads and giving the gateway time to recover.

---

### 2. Retry Pattern

**Analogy:** Pressing a button again when the elevator doesn't respond the first time.

#### How It Works

When a call fails due to a **transient** error (network blip, momentary overload), the Retry Pattern automatically re-attempts the operation a configurable number of times before giving up.

A naive retry immediately re-sends the request. A smarter retry uses **exponential backoff with jitter**:

```
Attempt 1  → fails → wait 1s
Attempt 2  → fails → wait 2s + random jitter
Attempt 3  → fails → wait 4s + random jitter
Attempt 4  → give up, throw exception
```

The jitter (randomness) is critical — without it, all retrying clients synchronize and create **thundering herd** spikes against the recovering service.

#### Purpose

- Handles transient, self-resolving failures transparently
- Reduces noise from brief network interruptions
- Avoids surfacing false errors to users for problems that resolve in milliseconds

#### ⚠️ Important Caveat

Retry must only be applied to **idempotent** operations. Retrying a `POST /orders` endpoint could create duplicate orders. Always verify the operation can safely be repeated before enabling retries.

---

### 3. Fallback Pattern

**Analogy:** If your GPS loses signal, you fall back to a paper map.

#### How It Works

When a primary operation fails (after retries are exhausted, or when the circuit is open), the Fallback Pattern kicks in with an **alternative response path**:

```
Primary Call
     │
     ▼
  Fails?
     │
     ▼
Fallback Strategy
  ├── Cached response
  ├── Default / static value
  ├── Alternative service
  └── Graceful "unavailable" message
```

#### Common Fallback Strategies

| Strategy | Description | Example |
|----------|-------------|---------|
| **Cached data** | Return last known good data | Serve yesterday's product pricing |
| **Static default** | Return a safe, hardcoded value | Return an empty recommendations list |
| **Alternate service** | Route to a backup provider | Use secondary SMS provider |
| **Fail silent** | Omit the feature entirely | Hide a widget if its API is down |

#### Purpose

- Maintains partial functionality during outages
- Prevents a non-critical dependency failure from breaking the entire user experience
- Enables informed, intentional degradation rather than surprise crashes

---

### 4. Bulkhead Pattern

**Analogy:** The watertight compartments in a ship's hull — flooding one section doesn't sink the whole vessel.

#### How It Works

The Bulkhead Pattern **isolates** different workloads into separate resource pools (thread pools, connection pools, process groups) so that excessive load or failure in one area cannot exhaust resources needed by another.

```
┌──────────────────────────────────────────┐
│               Thread Pool                │
│  ┌─────────────┐    ┌─────────────────┐  │
│  │  Service A  │    │   Service B     │  │
│  │  Pool: 20   │    │   Pool: 10      │  │
│  │  threads    │    │   threads       │  │
│  └─────────────┘    └─────────────────┘  │
└──────────────────────────────────────────┘
```

If Service A becomes slow and consumes all 20 of its threads, Service B's 10 threads remain completely unaffected.

#### Two Common Forms

- **Thread Pool Bulkhead:** Separate thread pools per downstream dependency
- **Semaphore Bulkhead:** Limit concurrent calls to each service using a counter (lighter-weight, no new threads)

#### Purpose

- Contains blast radius of failures and slowdowns
- Critical services remain responsive even when others are overwhelmed
- Prevents a "noisy neighbor" dependency from starving the entire application of resources

---

### 5. Timeout Pattern

**Analogy:** Hanging up the phone after waiting too long instead of holding indefinitely.

#### How It Works

Every call to an external service is given a **maximum time budget**. If the call doesn't complete within that window, it is cancelled and an error is returned.

```
t=0ms    → Call initiated
t=500ms  → Still waiting...
t=1000ms → Timeout threshold reached
t=1001ms → Call cancelled, exception thrown
           → Fallback or error response returned
```

Timeouts are typically configured at multiple levels:

| Level | Example |
|-------|---------|
| **Connection timeout** | How long to wait to establish a connection |
| **Read timeout** | How long to wait for data after connecting |
| **Circuit timeout** | How long the circuit stays open before testing again |
| **Overall request timeout** | End-to-end budget for the entire operation |

#### Purpose

- Frees threads and resources that would otherwise be blocked indefinitely
- Provides bounded, predictable latency for callers
- Works synergistically with Circuit Breakers — repeated timeouts can trip the breaker

#### ⚠️ Choosing Timeout Values

Too short → false failures under normal load spikes  
Too long → threads stay blocked, defeating the purpose

Base timeout values on measured **p99 latency** of healthy service responses, with a reasonable safety margin.

---

## How the Patterns Work Together

These patterns are not mutually exclusive — they are designed to **compose**:

```
Incoming Request
       │
       ▼
  ┌─────────────────────────────────────────────┐
  │              Bulkhead                        │
  │  (dedicated thread pool per dependency)      │
  │                                             │
  │   ┌──────────────────────────────────────┐  │
  │   │         Circuit Breaker              │  │
  │   │  ┌────────────────────────────────┐  │  │
  │   │  │   Retry + Timeout              │  │  │
  │   │  │   (with exponential backoff)   │  │  │
  │   │  └────────────────────────────────┘  │  │
  │   └──────────────────────────────────────┘  │
  │                    │                        │
  │              Fallback                       │
  │        (if all else fails)                  │
  └─────────────────────────────────────────────┘
```

A well-resilient service call:
1. Runs in an **isolated thread pool** (Bulkhead)
2. Is skipped immediately if the circuit is **open** (Circuit Breaker)
3. Has a bounded **time limit** (Timeout)
4. Is **retried** on transient failure with backoff (Retry)
5. Returns a safe alternative if ultimately unavailable (Fallback)

---

## Quick Reference

| Pattern | Protects Against | Key Mechanism |
|---------|-----------------|---------------|
| **Circuit Breaker** | Cascading failures, thundering herd | State machine that short-circuits calls to failing services |
| **Retry** | Transient / momentary failures | Re-attempt with backoff + jitter |
| **Fallback** | Non-critical dependency outages | Alternative response when primary fails |
| **Bulkhead** | Resource exhaustion by noisy neighbors | Isolated resource pools per dependency |
| **Timeout** | Indefinite blocking on slow services | Cancel calls that exceed a time budget |

---

*Resilience is not about avoiding failure — it's about designing systems that make failure boring.*