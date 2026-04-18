# The Event Loop

---

## Technical Explanation

The **event loop** is the mechanism that allows a single-threaded runtime to handle concurrency — particularly I/O-bound work — without blocking.

### The Core Problem

A single thread can only do one thing at a time. If it blocks waiting for a network response or disk read, everything else stalls. The event loop solves this by saying: *instead of waiting, register a callback and move on.*

### How It Works (Conceptually)

```
while (true) {
    task = queue.poll()
    if (task != null) task.run()
}
```

That's the skeleton. In practice there are multiple queues with different priorities, but the idea is the same: **pick up a task, run it to completion, pick up the next one.**

The key contract: **tasks must not block.** If a task blocks the thread, the whole loop stalls.

---

### The Two Worlds

**JavaScript (Node.js / Browser)**

The event loop is baked into the runtime. The call stack, the Web APIs / libuv thread pool, and the callback queues (macro-task queue + microtask queue) work together:

1. Call stack runs synchronous code
2. Async ops (`setTimeout`, `fetch`, `fs.readFile`) are offloaded to the platform
3. When they complete, callbacks are pushed to the queue
4. Once the call stack is empty, the loop drains the microtask queue first (Promises), then pulls the next macro-task (`setTimeout`, I/O callbacks)

**Java (Project Loom / Netty / Vert.x)**

Java's traditional thread-per-request model is the *opposite* of an event loop. But:

- **Netty / Vert.x** implement their own event loop on top of NIO — one thread handles many connections via `Selector`
- **Project Loom (Java 21+)** takes a different angle: virtual threads are so cheap that you *can* block them freely, and the JVM's scheduler remaps them onto carrier threads — giving you event-loop-like throughput without inversion-of-control style callbacks

---

### Microtasks vs Macrotasks (JS)

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");

// Output: 1, 4, 3, 2
```

`Promise.then` (microtask) drains *before* `setTimeout` (macrotask) even though both were scheduled "immediately." The microtask queue is fully flushed after each task before the next macro-task runs.

---

### What Blocks the Event Loop (Common Mistakes)

- CPU-heavy computation on the main thread (sorting a huge array, crypto, image processing)
- Synchronous file/network calls (`fs.readFileSync`, JDBC in a Netty handler)
- Infinite or very long loops

**Fix:** offload to a worker thread / thread pool, or break work into chunks.

---

## Plain English Explanation

Imagine a restaurant with **one waiter**.

A bad waiter would take your order, walk to the kitchen, and just *stand there staring at the wall* until your food is ready — then bring it to you. While he's waiting, no one else in the restaurant gets served.

A smart waiter takes your order, hands it to the kitchen, and immediately goes to take another table's order. When the kitchen rings a bell saying your food is ready, he picks it up and brings it over.

**That smart waiter is the event loop.**

---

Your computer program is the waiter. "Waiting for a database response" or "waiting for a file to load" is the kitchen cooking. Instead of freezing and staring at the wall, the program says *"let me know when that's done"* and goes off to handle other things. When the result is ready, it gets a notification and picks up where it left off.

---

The one rule: **the waiter can't get stuck doing something long himself.** If he personally spends 10 minutes folding napkins, every table still waits. Same with code — if you do heavy work directly on the loop, everything else freezes anyway.

---

*Everything else — microtasks, queues, selectors — is just the details of how the "kitchen bell" system is organized under the hood.*
