# LinkedIn's Real-Time Platform: A Deep Dive

> Based on a talk by **Akhilesh Gupta** from LinkedIn's engineering team at QCon London 2020, and a plain-English breakdown of the key technologies involved.

---

## Table of Contents

1. [The Talk: LinkedIn's Real-Time Platform](#1-the-talk-linkedins-real-time-platform)
2. [What is Server-Sent Events (SSE)?](#2-what-is-server-sent-events-sse)
3. [What is HTTP Long Polling?](#3-what-is-http-long-polling)

---

## 1. The Talk: LinkedIn's Real-Time Platform

### Background

The largest live stream in history was the Cricket World Cup semifinal between India and New Zealand, with over **25 million concurrent viewers** and **100 million total viewers**. The second largest was the British Royal Wedding with 80 million concurrent viewers.

Akhilesh Gupta's team at LinkedIn — the **Unreal Real-Time Team** — built a platform capable of handling interactions at this scale. Their guiding principle throughout:

> *"Problems in distributed systems can be solved by starting small. Solve the first problem, and add simple layers in your architecture to solve bigger problems."*

---

### The Core Problem

When millions of people watch a live video, every "like" or comment needs to appear on everyone else's screen almost instantly. This requires the server to **push data to clients** — the opposite of how HTTP normally works, where clients ask and servers respond.

---

### Layer 1: The Delivery Pipe — Server-Sent Events

**The challenge:** How does the server send a like back to a client without the client asking for it?

**The solution:** An **HTTP long poll** with **Server-Sent Events (SSE)**.

- The client opens a regular HTTP connection to the server
- Instead of closing after responding, the server **keeps it open**
- Whenever a like or comment occurs, the server pushes a small chunk of data down the open pipe
- The client's `EventSource` interface picks it up and renders it on screen

The only technical difference from a normal HTTP request is one header:

```
Accept: text/event-stream
```

**Why SSE over WebSockets?**
- SSE is plain HTTP — it's never blocked by firewalls
- WebSockets use a different protocol (`ws://`) that corporate firewalls sometimes block
- SSE is sufficient because data only needs to flow *server → client*; sending a like back uses a separate normal HTTP request

---

### Layer 2: Managing Thousands of Connections — Akka Actors

**The challenge:** A server might hold 100,000 open connections. You can't have a dedicated CPU thread per connection — threads are expensive (1–8MB of stack each).

**The solution:** **Akka Actors**

An Akka Actor is a tiny, lightweight object with:
- **State** — in this case, one open SSE connection
- **Behavior** — when I receive a like, push it down my connection
- **A mailbox** — a queue of messages waiting to be processed
- **Lazy thread assignment** — a thread is only assigned when there's something to process

Because actors are so cheap, you can have **millions of them** managed by a small pool of threads. A single machine can handle **100,000 open connections** this way.

A **supervisor actor** sits above all the individual connection actors and broadcasts messages down to the right ones.

> With 100,000 connections per machine, LinkedIn could serve all 18 million Royal Wedding viewers with just **180 machines**.

---

### Layer 3: Subscriptions — Sending the Right Like to the Right Person

**The challenge:** If you blast every like to every connected user, people watching Video A get likes from Video B.

**The solution:** **Subscriptions**

When you open a live video, your app quietly sends a message:

```
"I am watching Live Video #42"
```

The server stores this in an **in-memory subscriptions table**. When a like comes in for Video #42, the server routes it only to the connections subscribed to that video.

The table lives in-memory because:
- It's local to that machine — only relevant for its own connections
- If the machine dies, the connections die too — nothing to recover

---

### Layer 4: Scaling to Multiple Machines — The Dispatcher

**The challenge:** One server handles 100,000 connections. A cricket match has 25 million viewers. You need many servers.

**The solution:** A **Real-Time Dispatcher** + multiple **Frontend Servers**

```
Viewer → HTTP → Likes Backend → Dispatcher → Frontend Servers → Clients
```

- **Frontend servers** hold the actual connections to users and manage local subscriptions
- **The dispatcher** sits in the middle, routing incoming events to the right frontend servers

When a like comes in, the dispatcher checks which frontend servers have users subscribed to that live video and forwards it only to those. Each frontend server handles the last mile via its local in-memory table and Akka Actors.

This **two-level fan-out** is efficient:
- Dispatcher only talks to a handful of frontend servers (small fan-out)
- Frontend servers handle individual users (large fan-out)
- Concerns are cleanly separated

---

### Layer 5: Multiple Dispatchers + Shared Storage

**The challenge:** A single dispatcher becomes a bottleneck at high event rates.

**The solution:** Multiple dispatcher nodes + a **shared key-value store**

- Add more dispatcher nodes for higher throughput
- The subscriptions table (which frontend servers are watching which videos) is pulled out into a **shared key-value store** (Redis, Couchbase, etc.)
- Any dispatcher can read from it independently

**Performance numbers:**
- One dispatcher handles ~**5,000 events/second**
- With 10 dispatchers: **50,000 events/second** reaching frontend servers
- Frontend servers then fan this out further → **millions of likes/second** delivered

---

### Layer 6: Multiple Data Centers

**The challenge:** LinkedIn operates globally. A viewer in India might connect to a server in Asia, but a like event might originate in the US.

**The solution:** Keep subscriptions local to each data center; **fan out publishes globally**

When a like is published in Data Center 1:

1. The local dispatcher checks for local subscriptions
2. If no local viewers are watching that video, it forwards the like to **all peer dispatchers** in other data centers
3. Each peer dispatcher checks its own local subscriptions and delivers to local viewers

This is better than cross-data-center subscriptions because you'd need to constantly sync subscription data across the globe, which is complex and slow.

---

### The Full End-to-End Flow

```
1. You tap Like on a live cricket match
2. Your app → HTTP request → Likes Backend
3. Likes Backend → HTTP request → Dispatcher
4. Dispatcher → key-value store lookup → which frontend servers are subscribed?
5. Dispatcher → HTTP request → relevant Frontend Servers
6. Frontend Server → in-memory lookup → which connections are watching this video?
7. Akka Actors → push like down SSE connections
8. Your screen shows the like

Total time: ~75ms at p90
```

---

### Performance Summary

| Metric | Value |
|---|---|
| Connections per frontend machine | 100,000 |
| Machines needed for 18M viewers | ~180 |
| Events per dispatcher per second | ~5,000 |
| End-to-end latency (p90) | ~75ms |
| Effective likes/second (distributed) | Millions |

---

### What Else Runs on This Platform

Because the system is generic (it routes events to subscribers), LinkedIn uses it for:

- Likes and comments on live videos
- Typing indicators in messaging
- Seen receipts
- All of LinkedIn's instant messaging
- **Presence** — the green "online" dot next to people's names

---

### Design Tradeoffs

**Speed over guaranteed delivery.** If a dispatcher crashes mid-flight, some events may be dropped. For social interactions on live video, "fast and almost always delivered" is acceptable. LinkedIn monitors drop rates closely and investigates when they rise.

**Why not Kafka?** Kafka would provide delivery guarantees, but each frontend server would need to consume every live video topic — and adding more frontend servers wouldn't help because each new one still needs to consume everything. The fan-out doesn't scale.

---

### Key Takeaways

- Real-time content delivery enables powerful user interactions: likes, comments, polls, and discussions
- Use **SSE** for the persistent delivery pipe — it's simple, works everywhere, and is firewall-friendly
- Use **Akka Actors** (or equivalent event-driven frameworks) to manage millions of connections efficiently
- When you hit a limit, **horizontal scaling** is usually the answer — add a machine
- The same platform can be built with Node.js, Python, or any server framework; Redis or MongoDB for the key-value store
- The guiding principle: **start small, add layers**

---

## 2. What is Server-Sent Events (SSE)?

### The Core Idea

SSE is a technology that lets a server **push data to a client over a single, long-lived HTTP connection** — without the client repeatedly asking for updates.

---

### The Normal Web vs. SSE

**Normal HTTP (polling):**
```
Client: "Any new likes?" → Server: "No." → connection closes
  ... 1 second later ...
Client: "Any new likes?" → Server: "No." → connection closes
  ... 1 second later ...
Client: "Any new likes?" → Server: "Yes!" → connection closes
```

Wasteful — most requests get a "no" answer, and there's always a delay.

**SSE:**
```
Client: "Send me updates whenever you have them."
Server: "Sure." → keeps connection open
  ... 3 seconds later, a like happens ...
Server pushes: {"type": "like", "count": 42}
  ... a comment happens ...
Server pushes: {"type": "comment", "text": "Great match!"}
[Same connection, no reconnecting]
```

---

### How It Works

**The request:**
```http
GET /live-updates HTTP/1.1
Host: example.com
Accept: text/event-stream
```

**The response:**
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type": "like", "count": 42}

data: {"type": "comment", "text": "Great match!"}

data: {"type": "like", "count": 43}
```

The connection never closes. The server just keeps streaming `data:` blocks whenever something happens.

---

### Client Code (Browser)

```javascript
const source = new EventSource('/live-updates');

source.onmessage = function(event) {
    const data = JSON.parse(event.data);
    // render the like, comment, etc. on screen
};
```

That's it. The `EventSource` interface is **built into every modern browser natively** — no libraries needed. On Android and iOS, lightweight libraries provide the same interface.

---

### Server Code (Node.js Example)

```javascript
app.get('/live-updates', (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');

    // Push an event whenever something happens
    setInterval(() => {
        res.write(`data: ${JSON.stringify({ type: 'like' })}\n\n`);
    }, 1000);
});
```

---

### Key Characteristics

**One-way.** Data flows server → client only. The client sends data (e.g., clicking Like) via a separate normal HTTP request.

**Just HTTP.** SSE never leaves the HTTP protocol. Firewalls, proxies, and corporate networks treat it as normal web traffic. WebSockets use a different protocol that sometimes gets blocked.

**Auto-reconnects.** If the connection drops, the browser automatically reconnects. The `EventSource` interface also sends a `Last-Event-ID` header so the server can replay missed events.

**Named events.** You can label different event types:

```
event: like
data: {"count": 42}

event: comment
data: {"text": "Amazing!"}
```

And listen for them specifically:

```javascript
source.addEventListener('like', (e) => { /* handle like */ });
source.addEventListener('comment', (e) => { /* handle comment */ });
```

---

### SSE vs. WebSockets

| | SSE | WebSockets |
|---|---|---|
| Direction | Server → Client only | Bidirectional |
| Protocol | Plain HTTP | Upgraded to `ws://` |
| Firewall friendly | Yes | Sometimes blocked |
| Auto-reconnect | Built in | You build it yourself |
| Browser support | Native `EventSource` API | Native but more complex |
| Best for | Live feeds, notifications, updates | Chat, games, collaborative editing |

**Rule of thumb:** If your client mostly *receives* updates (live scores, social feeds, stock prices), SSE is simpler and more reliable. If your client needs to *send* data back rapidly and continuously (multiplayer game, collaborative document), WebSockets make more sense.

---

## 3. What is HTTP Long Polling?

### The Fundamental Problem

HTTP was designed as a request-response protocol: client asks, server answers, connection closes. Real-time systems need the server to say "something just happened" — without waiting for the client to ask first.

---

### Three Approaches Compared

**1. Regular Polling (naive)**

The client asks on a fixed timer:

```
Client → "Any updates?" → Server: "No." (closes)
  [1 second passes]
Client → "Any updates?" → Server: "No." (closes)
  [1 second passes]
Client → "Any updates?" → Server: "Yes!" (closes)
```

Works, but enormously wasteful. Most requests are wasted, and events are always delayed by up to one polling interval.

---

**2. Long Polling (clever workaround)**

The server holds the request open until it has data:

```
Client → "Any updates?" → Server: [holds connection open...]
  [3 seconds pass, a like happens]
Server: "Yes, here's one!" (closes)
Client immediately opens a new request → Server: [holds again...]
```

Updates arrive almost instantly because the server was already holding a connection ready to go. If nothing happens, the server sends an empty response after a timeout (~30 seconds) and the client reconnects.

---

**3. Persistent Connection / SSE (what LinkedIn uses)**

The connection never closes at all:

```
Client → "Send me updates" → Server: [keeps connection open forever]
  A like → Server pushes it
  A comment → Server pushes it
  Another like → Server pushes it
[Same connection, no reconnecting]
```

This is SSE on top of HTTP long polling. "Long poll" = connection stays open. SSE = structured streaming format for data flowing through it.

---

### What It Looks Like

**Normal HTTP request:**
```
GET /api/data HTTP/1.1

→ HTTP/1.1 200 OK
→ {"result": "some data"}
→ [connection closes in milliseconds]
```

**Long poll request:**
```
GET /api/updates HTTP/1.1

→ [server holds the connection]
→ [8 seconds pass, nothing happens]
→ [an event occurs]
→ HTTP/1.1 200 OK
→ {"type": "like", "count": 5}
→ [connection closes — client immediately opens a new one]
```

---

### The Server-Side Engineering Challenge

Holding thousands of open connections without a dedicated thread per connection is the hard part. A traditional threaded model would fail because:

- Each thread uses ~1–8MB of stack space
- 100,000 connections = potentially 100,000 threads
- That's gigabytes of memory doing nothing but waiting

The solution is an **event-driven, non-blocking model** (like Akka Actors) where a small thread pool serves millions of open connections — a thread is only assigned when there's something to process.

---

### The Relationship Between Long Polling and SSE

These terms are related but distinct:

| Term | What It Is |
|---|---|
| **HTTP Long Polling** | The *connection strategy* — keep an HTTP connection open rather than closing it |
| **Server-Sent Events (SSE)** | The *protocol on top* — a standardized format for streaming multiple events down that connection, with a browser API to handle them |

```
HTTP Long Poll = the open pipe
SSE            = the structured format of data flowing through that pipe
```

---

### Long Polling vs. WebSockets at the Connection Level

| | Long Polling | WebSockets |
|---|---|---|
| Protocol | Plain HTTP throughout | Starts as HTTP, upgrades to `ws://` |
| How it opens | Normal GET request | HTTP upgrade handshake |
| Direction | Primarily one-way | Fully bidirectional |
| Firewall behavior | Looks like normal web traffic | Can be blocked |
| Message overhead | Low (connection already open) | Very low (binary framing) |

WebSockets say: *"Let's switch from HTTP to a faster protocol."* Long polling never leaves HTTP — it just stretches the connection. That's why long polling is universally firewall-friendly.

---

### A Mental Model

| Approach | Analogy |
|---|---|
| Regular polling | Repeatedly calling someone to ask if they have news |
| Long polling | Calling and staying on the line until they have something to tell you |
| SSE | Tuning into a radio broadcast — you tune in once and listen to everything that comes through |

Each step reduces wasted effort and moves closer to true real-time delivery.

---