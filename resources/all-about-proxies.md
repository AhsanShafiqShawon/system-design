# Proxies, API Gateways & Load Balancers
> Plain-English explanations — with TLS termination, compression, buffering, and routing

---

## 1. What Is a Proxy?

A **proxy** is simply a middleman — a server that sits between two parties and passes traffic between them.

Instead of A talking directly to B, A talks to the proxy, and the proxy talks to B.

Why? Because the middleman can *do things* — inspect, filter, transform, cache, log, or route — while the two real parties don't even have to know the full picture.

```
Client  ──►  Proxy  ──►  Server
```

That's it. Every concept below is just a specialised version of this idea.

---

## 2. Forward Proxy

> *"I'll go fetch that for you."*

A **forward proxy** sits on the **client side**. The client is configured to send all its requests through the proxy. The destination server sees the proxy's IP, not the client's.

```
[Your Browser]
     │
     ▼
[Forward Proxy]   ◄── client-side middleman
     │
     ▼
[Internet / Servers]
```

### What it's used for

| Use case | What's happening |
|---|---|
| **Corporate firewall** | Company routes all employee traffic through a forward proxy to block social media or scan for malware |
| **VPN / anonymity** | Your real IP is hidden; the server only sees the proxy |
| **Caching** | Proxy caches popular resources so the same file isn't downloaded 1,000 times by 1,000 employees |
| **Geo-restriction bypass** | Request appears to come from the proxy's country, not yours |

### Key point
The **client knows about the proxy**. The **server does not** (it just sees a request from some IP it doesn't recognise as yours).

---

## 3. Reverse Proxy

> *"I'll handle that — don't bother the real server."*

A **reverse proxy** sits on the **server side**. Clients talk to it thinking it's the actual server. The proxy then decides which real backend server should handle the request.

```
[Clients on the Internet]
         │
         ▼
  [Reverse Proxy]   ◄── server-side middleman
    /    |    \
   ▼     ▼     ▼
 [App1] [App2] [App3]   ◄── real backend servers
```

### What it's used for

| Use case | What's happening |
|---|---|
| **Hide backend topology** | Clients never know how many servers you have or what IPs they are |
| **TLS termination** | Proxy handles HTTPS so backends only deal with plain HTTP internally *(more on this below)* |
| **Caching** | Serve static files from cache without hitting your app |
| **Compression** | Compress responses before sending to client |
| **Single entry point** | One IP/domain, many backend services |

### Key point
The **server knows about the proxy** (it's their proxy). The **client does not** — it thinks it's talking directly to `api.yourcompany.com`.

---

## 4. API Gateway

> *"You shall not pass — unless you're authenticated, rate-limited, and talking to the right service."*

An **API Gateway** is a reverse proxy that's **purpose-built for APIs**. It's not just a dumb traffic router — it's an intelligent gatekeeper.

```
[Mobile App]  [Web App]  [Third-party Client]
       \           |           /
        ▼          ▼          ▼
      ┌────────────────────────┐
      │       API Gateway      │
      │  • Auth / JWT check    │
      │  • Rate limiting       │
      │  • Request routing     │
      │  • Response transform  │
      │  • Logging / metrics   │
      └────────────┬───────────┘
           ┌───────┼────────┐
           ▼       ▼        ▼
      [User Svc] [Order] [Payment]
```

### What it adds on top of a plain reverse proxy

| Feature | What it means |
|---|---|
| **Authentication** | Check if the caller has a valid API key or JWT token |
| **Rate limiting** | "This client can only make 100 requests per minute" |
| **Request routing** | `/users/*` → User Service, `/orders/*` → Order Service |
| **Request/response transformation** | Rename fields, convert XML to JSON, strip internal headers |
| **Aggregation** | Call 3 services, stitch results, return one response |
| **Analytics** | Who called what, how often, with what latency |

### Key point
A plain reverse proxy routes traffic. An API Gateway **understands** that traffic — it knows about authentication, versioning, rate limits, and business rules.

Popular examples: **Kong**, **AWS API Gateway**, **NGINX with plugins**, **Apigee**.

---

## 5. Load Balancer

> *"I'll spread the work around so nobody burns out."*

A **load balancer** is a reverse proxy whose primary job is to **distribute traffic across multiple identical servers** so no single server is overwhelmed.

```
          [Clients]
              │
              ▼
      [Load Balancer]
       /      |      \
      ▼       ▼       ▼
  [Server1] [Server2] [Server3]
  (33%)      (33%)     (33%)
```

### Common distribution algorithms

| Algorithm | Behaviour |
|---|---|
| **Round Robin** | Server 1, Server 2, Server 3, Server 1, … (in rotation) |
| **Least Connections** | Always send to whichever server has the fewest active connections right now |
| **IP Hash** | Same client IP always goes to the same server (useful for sticky sessions) |
| **Weighted** | Server 1 gets 60%, Server 2 gets 40% (if one is more powerful) |

### What else it does

- **Health checks** — constantly pings servers; if one fails, it stops sending traffic there
- **Session stickiness** — optionally keeps a user on the same server for the duration of their session
- **TLS termination** — often handles HTTPS so backends don't have to *(same as reverse proxy)*

### Key point
Load balancers are about **resilience and scalability**. They're why you can take a server offline for maintenance without users noticing, and why you can handle 10× traffic by just adding more servers.

Popular examples: **AWS ELB/ALB**, **HAProxy**, **NGINX**, **Traefik**.

---

## 6. Supporting Concepts

### TLS Termination

HTTPS (TLS) is encrypted. Decrypting it is computationally expensive.

**TLS termination** means: let the proxy/gateway/load balancer decrypt the HTTPS traffic once, then forward plain HTTP to your backend servers. Your backend servers are relieved of crypto work, and you only manage one TLS certificate in one place.

```
Client  ──HTTPS──►  [Proxy decrypts here]  ──HTTP──►  Backend
                    ← TLS termination point →
```

The internal network (between proxy and backend) is assumed trusted, so plain HTTP is acceptable there. If you need end-to-end encryption even internally, that's called **TLS passthrough** or **re-encryption** — the proxy either leaves the traffic encrypted or re-encrypts it before forwarding.

---

### Compression

Responses (especially JSON, HTML, CSS) contain lots of repetitive text. Compressing them (usually with **gzip** or **Brotli**) can reduce payload size by 60–80%.

The proxy compresses the response before sending it to the client. The client decompresses it. This happens transparently — the application server doesn't have to do it.

```
[Backend]  ──raw JSON (100 KB)──►  [Proxy compresses]  ──gzip (18 KB)──►  [Client]
```

Benefit: faster responses, lower bandwidth costs, better experience on mobile networks.

---

### Buffering

Imagine a slow client on a 3G connection downloading a response. Without buffering, the backend server has to hold the connection open and drip-feed bytes to the slow client. The backend process is **stuck** the whole time.

With **buffering**, the proxy:
1. Receives the full response from the backend quickly
2. Releases the backend immediately
3. Trickles the response to the slow client at whatever pace it can handle

```
Backend  ──fast──►  [Proxy buffers full response]  ──slow──►  Slow Client
                     backend is free immediately
```

This frees your application servers from worrying about slow clients and allows them to handle more requests.

Buffering also works on the **request side** — the proxy waits for the full request body before forwarding to the backend, protecting it from slow uploads.

---

### Routing

Routing is the proxy's decision: *"Given this request, which backend should handle it?"*

There are several levels of routing sophistication:

| Type | Rule example |
|---|---|
| **Path-based** | `/api/users` → User Service, `/api/orders` → Order Service |
| **Host-based** | `app.example.com` → App Servers, `admin.example.com` → Admin Servers |
| **Header-based** | `X-Version: v2` → New backend, otherwise → Old backend |
| **Method-based** | `GET` requests → Read replicas, `POST/PUT` → Primary servers |
| **Canary / weighted** | 5% of traffic → new version, 95% → stable version |

More sophisticated routing logic generally lives in an **API Gateway** or a **service mesh**, while basic path/host routing is standard in any reverse proxy or load balancer.

---

## 7. Load Balancer Deep Dive

### L4 vs L7 — The Two Layers of Load Balancing

These labels come from the **OSI model** — a conceptual stack that describes how network communication is organised, from raw electrical signals at the bottom to application data at the top. You don't need to memorise all 7 layers. Just know two of them:

- **Layer 4 (Transport layer)** — knows about **TCP/UDP connections**: source IP, destination IP, port numbers. That's it. It cannot see inside the data being transferred.
- **Layer 7 (Application layer)** — knows about **HTTP/HTTPS**: URLs, headers, cookies, request bodies, response codes. It understands what the traffic actually *is*.

---

#### L4 Load Balancer

> *"I see a TCP connection arriving on port 443. I'll forward it to Server 2."*

An L4 load balancer works at the **connection level**. It makes routing decisions based purely on network metadata — IP address and port — without ever looking at the content of the packets.

```
Client TCP connection  ──►  [L4 LB]  ──►  picks a backend by IP:port
                                           (never opens the packet)
```

**How it works in practice:**

When a client opens a TCP connection to the load balancer, the LB picks a backend and creates a second TCP connection to that server. It then forwards raw bytes between the two, acting as a TCP relay. It never reads HTTP headers, URLs, or cookies.

**Characteristics:**

| Property | Detail |
|---|---|
| **Speed** | Extremely fast — minimal processing, no packet inspection |
| **Protocol agnostic** | Works with any TCP/UDP protocol: HTTP, gRPC, MySQL, SMTP, custom binary |
| **Routing intelligence** | Low — can only use IP, port, and connection-level info |
| **TLS** | Passes through encrypted traffic without decrypting it (TLS passthrough) |
| **Stickiness** | Only IP-hash based (same IP → same server) |

**When to use it:** High-throughput, low-latency scenarios — database connection pooling, real-time gaming, video streaming, any non-HTTP protocol.

---

#### L7 Load Balancer

> *"I see a `POST /api/bookings` request with a JWT token for user #42. I'll route it to the Bookings cluster and log the request."*

An L7 load balancer operates at the **HTTP level**. It fully decrypts and reads the request, makes intelligent routing decisions based on its content, and then forwards it (possibly modified) to a backend.

```
HTTP Request  ──►  [L7 LB decrypts + reads headers/URL/body]
                         │
                    routing decision
                    (path, host, header, cookie, method)
                         │
                         ▼
                    [Backend Server]
```

**Characteristics:**

| Property | Detail |
|---|---|
| **Routing intelligence** | High — path, host, headers, cookies, HTTP method, query params |
| **TLS termination** | Yes — decrypts HTTPS, so it can read the content |
| **Sticky sessions** | Cookie-based (much more reliable than IP hash) |
| **Request manipulation** | Can add/remove headers, rewrite URLs, inject tracing IDs |
| **Protocol** | HTTP/1.1, HTTP/2, gRPC, WebSocket |
| **Speed** | Slightly slower than L4 due to packet inspection — still very fast in practice |

**When to use it:** Web applications, REST APIs, microservices — anywhere you want smart routing, TLS termination, or request-level observability.

---

#### L4 vs L7 — Side by Side

```
                    L4 Load Balancer        L7 Load Balancer
                   ─────────────────       ──────────────────
Sees:              IP + Port               Full HTTP request
TLS:               Passthrough             Terminates (decrypts)
Routing by:        IP, port                URL, headers, cookies
Protocols:         Any TCP/UDP             HTTP, gRPC, WebSocket
Speed:             ████████████ faster     ████████░░░░ slightly slower
Intelligence:      ████░░░░░░░░ low        ████████████ high
Use case:          DB, TCP, streaming      Web APIs, microservices
```

In practice, most web applications use an **L7 load balancer** (like AWS ALB, NGINX, Traefik). L4 load balancers (like AWS NLB) appear when you need raw throughput, non-HTTP protocols, or to preserve the client's original IP address.

---

### Health Checks

A load balancer is only useful if it's sending traffic to servers that are **actually working**. Health checks are how it finds out.

The load balancer periodically probes each backend server and marks it as **healthy** or **unhealthy**. Traffic only goes to healthy servers.

#### Types of health checks

**Passive (implicit)** — The LB watches real traffic. If a server returns too many errors (5xx) or times out repeatedly, it marks it unhealthy. No extra probes. Downside: a real user request has to fail first before the LB notices.

**Active (explicit)** — The LB sends dedicated probe requests on a schedule, independent of real traffic. This is the standard approach.

Active checks come in levels of sophistication:

| Level | What's checked | Example |
|---|---|---|
| **TCP ping** | Can I open a TCP connection to port 8080? | Server process is up and listening |
| **HTTP check** | Does `GET /health` return 200 OK? | App is responding |
| **Deep health check** | Does `/health` confirm DB connection, cache, and dependencies are up? | App is *truly* ready |

A well-designed deep health check endpoint in your app might look like this:

```
GET /health/ready

{
  "status": "UP",
  "db": "UP",
  "cache": "UP",
  "diskSpace": "UP"
}
→ returns 200 OK   ✓ LB sends traffic here

{
  "status": "DOWN",
  "db": "DOWN"     ← can't reach database
}
→ returns 503   ✗ LB stops sending traffic here
```

#### Health check configuration

```
interval:           10s    ← probe every 10 seconds
timeout:            2s     ← if no response in 2s, count as failed
healthy_threshold:  2      ← 2 consecutive successes → mark healthy
unhealthy_threshold: 3     ← 3 consecutive failures  → mark unhealthy
```

The **thresholds** prevent flapping — a server isn't pulled out of rotation because of one transient hiccup, and it isn't put back until it's proven stable.

#### What happens when a server goes unhealthy

```
[LB] ──► Server1 ✓  (healthy, gets traffic)
     ──► Server2 ✓  (healthy, gets traffic)
     ──► Server3 ✗  (unhealthy — LB stops routing here)

Server3 later recovers → passes health checks → LB re-adds it to rotation
```

This is the mechanism that makes zero-downtime deployments possible. When you deploy a new version of your app, you can take servers out of rotation gracefully, update them, wait for health checks to pass, and bring them back — all without a single dropped request.

---

### Connection Draining

> *"Before I take this server offline, let me wait for it to finish what it's doing."*

When you want to remove a server from the pool — for deployment, scaling down, or maintenance — you can't just cut the connection. There might be **in-flight requests** on that server right now: a user uploading a booking, a payment being processed, a report being generated. Kill the server abruptly and those requests fail.

**Connection draining** (also called **deregistration delay** in AWS) is the graceful shutdown process:

```
Normal operation:
LB  ──► Server A  ✓ (gets new connections)
    ──► Server B  ✓ (gets new connections)
    ──► Server C  ✓ (gets new connections)

Server C marked for removal:
LB  ──► Server A  ✓ (gets new connections)
    ──► Server B  ✓ (gets new connections)
    ──► Server C  ⏳ DRAINING
                      ├── existing request #1... completing
                      ├── existing request #2... completing
                      └── no new requests accepted

After drain timeout (e.g. 30s):
    ──► Server C  ✗ removed — now safe to shut down or redeploy
```

#### The drain window

You configure a **drain timeout** — a maximum time to wait. Typical values are 30–300 seconds depending on how long your longest requests might take.

- Short-lived requests (simple REST APIs): 30 seconds is plenty
- Long-running requests (file uploads, report generation, hotel search aggregation): set it longer

If requests finish before the timeout, the server is freed immediately. If any requests are still running when the timeout expires, they are forcibly terminated — so set the timeout conservatively.

#### Connection draining in a deployment pipeline

This is what a **zero-downtime rolling deployment** actually looks like step by step:

```
1. Signal Server C to start draining
   → LB stops sending new requests to Server C

2. Wait for drain (existing requests finish)

3. Server C is now idle — deploy new version

4. New version starts, passes health checks

5. LB adds Server C back to rotation

6. Repeat for Server A, then Server B
```

At no point is your service unavailable. Users never see an error. This is why connection draining is a prerequisite for safe, professional deployments.

#### In Spring Boot

Spring Boot has built-in graceful shutdown support. When the JVM receives a `SIGTERM` (which your orchestrator or deployment script sends), Spring Boot:

1. Stops accepting new HTTP requests
2. Waits for in-flight requests to complete (up to a configured timeout)
3. Then shuts down

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

This works hand-in-hand with the load balancer's connection draining — the LB drains at the network level, and Spring drains at the application level. Together they guarantee no request is dropped on deployment.

---

## 9. How They All Relate

Here's the big picture, laid out plainly:

```
[User's Browser / Mobile App]
         │
         │  HTTPS
         ▼
┌─────────────────────────────┐
│        API Gateway          │  ← Authenticates, rate-limits,
│   (is a reverse proxy +)    │    routes to the right service
└──────────────┬──────────────┘
               │ HTTP (TLS terminated here)
               ▼
┌─────────────────────────────┐
│       Load Balancer         │  ← Spreads load, health checks,
│   (is a reverse proxy +)    │    ensures no single server dies
└───────┬──────────┬──────────┘
        │          │
        ▼          ▼
  [App Server 1]  [App Server 2]   ← Your actual application
```

And if your employees need to reach an external service:

```
[Employee's Machine]
        │
        ▼
[Forward Proxy]   ← Enforces company policy, hides internal IPs
        │
        ▼
[External Internet / Third-party API]
```

### One-sentence summary of each

| Concept | One sentence |
|---|---|
| **Proxy** | A middleman that passes traffic between two parties |
| **Forward Proxy** | A client-side middleman that hides the client from the server |
| **Reverse Proxy** | A server-side middleman that hides backend servers from clients |
| **API Gateway** | A smart reverse proxy that enforces auth, rate limits, and business routing rules for APIs |
| **Load Balancer** | A reverse proxy whose main job is distributing traffic across many identical servers |
| **TLS Termination** | Decrypting HTTPS at the proxy so backends handle plain HTTP |
| **Compression** | Shrinking responses at the proxy before sending to clients |
| **Buffering** | Absorbing slow clients at the proxy so backends are freed up fast |
| **Routing** | Deciding which backend handles a request based on path, headers, method, or weight |
| **L4 Load Balancer** | Routes TCP connections by IP/port — fast, protocol-agnostic, no content inspection |
| **L7 Load Balancer** | Routes HTTP requests by URL/headers/cookies — intelligent, inspects content, does TLS termination |
| **Health Check** | Periodic probes that tell the LB which servers are alive and ready to take traffic |
| **Connection Draining** | Graceful removal of a server — stops new traffic, waits for in-flight requests to finish |

---