# The Request-Response Pipeline: A Deep Dive

> What really happens between the moment you press Enter and the moment a webpage appears on your screen? This document traces the full journey of a request — from your browser all the way to a backend server and back — explaining every major layer along the way.

---

## Table of Contents

1. [What Happens When You Type `google.com`?](#1-what-happens-when-you-type-googlecom)
2. [What Is an API Gateway?](#2-what-is-an-api-gateway)
3. [What Is a Load Balancer?](#3-what-is-a-load-balancer)
4. [What Is a Reverse Proxy?](#4-what-is-a-reverse-proxy)
5. [Putting It All Together: The Full Pipeline](#5-putting-it-all-together-the-full-pipeline)

---

## 1. What Happens When You Type `google.com`?

Before your browser can send a single byte of HTTP traffic to Google's servers, it needs to answer one fundamental question: **what is Google's IP address?** The internet doesn't work with domain names — it works with IP addresses. The process of translating `google.com` → `142.250.190.46` (or similar) is called **DNS Resolution**, and it involves several layers of caching and lookup.

### Step 1 — Browser Cache

The very first thing your browser does is check **its own internal DNS cache**. Browsers keep a short-lived record of domain-to-IP mappings from previous visits. If you visited `google.com` five minutes ago, the answer is probably still cached here. The browser checks the TTL (Time To Live) attached to that record — if it hasn't expired, it uses the cached IP immediately and skips all remaining DNS steps.

You can inspect Chrome's DNS cache at `chrome://net-internals/#dns`.

### Step 2 — OS Cache

If the browser cache misses, the browser asks the **operating system** to resolve the domain. The OS also maintains its own DNS cache (managed by a local resolver daemon like `nscd` on Linux or the DNS Client service on Windows). Before making any network request, the OS checks this cache.

The OS also checks the **`/etc/hosts` file** (or `C:\Windows\System32\drivers\etc\hosts` on Windows) — a plain-text file that maps hostnames to IPs locally, overriding all DNS lookups. This is why developers often add entries like `127.0.0.1 myapp.local` for local development.

### Step 3 — Router-Level Cache

If the OS cache also misses, the query travels to your **local router** (your home gateway). Most routers have a small built-in DNS cache for commonly visited domains. If another device on your network recently visited `google.com`, the router might have the answer cached and responds immediately without going further.

### Step 4 — Recursive Resolver (Your ISP or a Public DNS)

If the router has no cached answer, it forwards the query to a **Recursive Resolver** — typically one provided by your ISP (Internet Service Provider), or a public one like Google's `8.8.8.8` or Cloudflare's `1.1.1.1`.

The Recursive Resolver is the workhorse of DNS. It doesn't just look things up once — it **recursively traverses the DNS hierarchy** on your behalf:

1. It first asks a **Root Name Server** (there are 13 logical root servers worldwide): *"Who knows about `.com` domains?"*
2. The root server points it to a **TLD (Top-Level Domain) Name Server** for `.com`.
3. The Recursive Resolver then asks the TLD server: *"Who knows about `google.com`?"*
4. The TLD server points to Google's own **Authoritative Name Server**.
5. The Recursive Resolver asks Google's Authoritative Name Server: *"What is the IP for `google.com`?"*
6. The Authoritative Name Server responds with the final IP address.

The Recursive Resolver caches this result according to the record's TTL and returns the IP to your OS, which caches it too, which passes it to the browser, which caches it too.

### Step 5 — Authoritative Name Server

The **Authoritative Name Server** is the definitive, ground-truth source for a domain's DNS records. It is configured and managed by the domain owner (in this case, Google). It holds the actual `A` records (IPv4), `AAAA` records (IPv6), `CNAME` records (aliases), `MX` records (mail), and so on.

Once the Recursive Resolver gets the IP from the Authoritative Name Server, the DNS resolution phase is complete.

### Step 6 — TCP Handshake + TLS Handshake

Now that your browser has an IP, it opens a **TCP connection** (3-way handshake: SYN → SYN-ACK → ACK). For HTTPS (which all modern sites use), a **TLS handshake** follows immediately — negotiating encryption keys and verifying the server's certificate.

### Step 7 — HTTP Request

With a secure connection established, the browser sends an **HTTP GET request** for the page. The request travels through the network infrastructure described in the rest of this document.

---

## 2. What Is an API Gateway?

Once a request reaches your system's network boundary, the very first thing it typically encounters is an **API Gateway** — a single entry point that every inbound request must pass through before reaching any backend service.

Think of it as the **front desk of a large office building**. You don't walk straight to someone's desk. You go through reception, show your ID, state your purpose, get directed to the right floor, and possibly have your bag scanned. The API Gateway does all of this for HTTP traffic.

### Authentication & Authorization

The API Gateway verifies **who is making the request** and **whether they are allowed to do what they're asking**.

- It validates **API keys**, **JWT tokens**, **OAuth2 bearer tokens**, or **session cookies**.
- It can integrate with an Identity Provider (IdP) like Auth0, Keycloak, or AWS Cognito.
- If authentication fails, the gateway immediately returns a `401 Unauthorized` or `403 Forbidden` response — the request never reaches your backend servers at all. This is an important security boundary.

### Rate Limiting

The gateway enforces **how many requests a client is allowed to make** in a given time window. This protects backend services from being overwhelmed, whether by a misbehaving client, a buggy retry loop, or a deliberate DDoS attack.

Examples:
- *"This API key is allowed 1,000 requests per minute."*
- *"This IP address has exceeded 100 requests per second — return `429 Too Many Requests`."*

Rate limiting can be applied globally, per endpoint, per user, per API key, or per IP.

### Routing

The API Gateway inspects the request path and method and **routes it to the correct backend service**. This is especially important in a microservices architecture where you might have dozens of independent services:

- `GET /users/123` → User Service
- `POST /orders` → Order Service
- `GET /search?q=hotels` → Search Service

The gateway acts as a **single URL namespace** for your entire platform. Clients never need to know where individual services live.

### Adding Metadata

Before forwarding a request downstream, the gateway often **enriches it with additional context** — attaching information that backends can use without having to re-derive it themselves:

- `X-User-ID: 98765` — the authenticated user's ID (derived from the JWT)
- `X-Request-ID: abc-123-xyz` — a unique trace ID for distributed tracing
- `X-Forwarded-For: 203.0.113.5` — the original client IP address
- `X-Tenant-ID: acme-corp` — the tenant in a multi-tenant system

This way, backends receive pre-verified, pre-enriched requests and don't each need to implement authentication logic independently.

### Other Common Responsibilities

- **Request/Response transformation** — rewriting headers, converting XML to JSON, etc.
- **SSL termination** — decrypting HTTPS traffic so backends can speak plain HTTP internally
- **Caching** — returning cached responses for idempotent requests
- **Logging & monitoring** — capturing every request for observability

---

## 3. What Is a Load Balancer?

After the API Gateway has authenticated and routed the request, it typically arrives at a **Load Balancer**. Where the API Gateway is concerned with *what* the request is, the Load Balancer is concerned with *who handles it*.

The core problem: you have many identical servers (instances) capable of handling a request, but the client is only talking to one IP address. The Load Balancer sits between the client-facing IP and the pool of servers, **distributing incoming requests across multiple backend instances** so that no single server becomes a bottleneck.

### Health Checks

A Load Balancer continuously monitors the health of every server in its pool. It regularly sends **health check probes** — typically a lightweight HTTP request to a `/health` or `/ping` endpoint — to each backend instance.

If a server fails to respond, responds too slowly, or returns an error status:
- The Load Balancer **marks it as unhealthy** and stops sending new traffic to it.
- If the server recovers and starts passing health checks again, it is automatically **re-added to the pool**.

This means a deployment crash, an out-of-memory error, or a hung process on one server will not take down your entire application — the Load Balancer silently reroutes around it.

### Traffic Distribution Algorithms

Load balancers use different strategies to decide which server gets each request:

- **Round Robin** — requests are distributed to each server in turn, one at a time. Simple and fair when all servers are identical and all requests have similar cost.
- **Least Connections** — the request goes to whichever server currently has the fewest active connections. Better when requests vary significantly in processing time.
- **IP Hash** — the client's IP address is hashed to consistently route that client to the same backend server. Useful for stateful applications where session affinity matters.
- **Weighted Round Robin** — like Round Robin, but servers with more capacity get proportionally more traffic.

### Layer 4 vs. Layer 7 Load Balancing

- **Layer 4 (Transport Layer)** — operates at the TCP/UDP level. Fast and low-overhead, but can only route based on IP and port. Does not inspect HTTP content.
- **Layer 7 (Application Layer)** — operates at the HTTP level. Can route based on URL path, HTTP headers, cookies, or even request body content. More powerful but slightly more expensive computationally.

Modern load balancers like AWS ALB (Application Load Balancer), NGINX, and HAProxy operate at Layer 7.

---

## 4. What Is a Reverse Proxy?

A **Reverse Proxy** sits in front of one or more backend servers and **acts on their behalf** toward the outside world. Clients talk to the reverse proxy — they often don't even know (or need to know) that backend servers exist.

The "reverse" in the name distinguishes it from a **forward proxy**, which sits in front of clients and acts on their behalf toward the outside world (like a corporate firewall or VPN). A reverse proxy does the mirror image: it represents servers to clients.

### HTTPS to HTTP Termination (SSL Offloading)

Backend servers often communicate over plain HTTP on a private internal network — there's no need to encrypt traffic between machines in the same data center. But all public-facing traffic should be encrypted with HTTPS.

The Reverse Proxy handles this translation:

1. The client connects via **HTTPS** (encrypted, TLS).
2. The Reverse Proxy decrypts the traffic (SSL termination).
3. It forwards the request to the backend over **plain HTTP**.
4. The response comes back over plain HTTP, the Reverse Proxy encrypts it, and sends it back to the client over HTTPS.

This offloads the CPU-intensive work of TLS encryption/decryption from your backend servers. It also means you manage SSL certificates in one place (the reverse proxy) rather than on every individual server.

### Compression

Reverse proxies can **compress response bodies** (using gzip or Brotli) before sending them to the client. A 500KB JSON payload might compress down to 80KB, significantly reducing bandwidth usage and improving perceived page load time — especially for mobile users on slower connections.

Compression can be applied selectively: large text-based responses (JSON, HTML, CSS, JS) compress well; binary content like images and videos generally does not.

### Chunked Transfer (Streaming Responses)

For large responses — a file download, a long database result, a video stream — it's impractical to buffer the entire response in memory before sending it. Reverse proxies support **chunked transfer encoding**, which means:

1. The backend starts generating the response.
2. As chunks of data become available, the Reverse Proxy **streams them to the client immediately**, without waiting for the full response.
3. The client begins receiving and processing data before the server has finished producing it.

This is how streaming APIs (think ChatGPT's token-by-token output), large file downloads, and Server-Sent Events work efficiently.

### Other Responsibilities

- **Caching** — storing responses to common requests so the backend doesn't need to recompute them
- **Request buffering** — absorbing slow client uploads so the backend receives the complete request at once
- **Security** — hiding internal server topology, absorbing malformed requests before they reach application code
- **Serving static assets** — offloading delivery of CSS, JS, and images from the application server entirely

Popular reverse proxies include **NGINX**, **Caddy**, **HAProxy**, and **AWS CloudFront**.

---

## 5. Putting It All Together: The Full Pipeline

Here is the complete journey of a request — from the moment you press Enter to the moment you see a response rendered in your browser.

```
Your Browser
     │
     │  1. DNS Resolution
     │     ├── Browser Cache
     │     ├── OS Cache + /etc/hosts
     │     ├── Router Cache
     │     ├── Recursive Resolver (ISP / 8.8.8.8 / 1.1.1.1)
     │     └── Authoritative Name Server  →  IP Address resolved
     │
     │  2. TCP + TLS Handshake (connection established)
     │
     ▼
[ Reverse Proxy ]
     │  - Terminates HTTPS → forwards as HTTP internally
     │  - Will compress the response before returning it
     │  - Will stream response in chunks if large
     │
     ▼
[ API Gateway ]
     │  - Authenticates the request (JWT / API key / OAuth)
     │  - Rate limits the client
     │  - Routes to the correct backend service
     │  - Injects metadata (User-ID, Request-ID, Tenant-ID)
     │
     ▼
[ Load Balancer ]
     │  - Health-checks all backend instances
     │  - Picks a healthy instance (Round Robin / Least Conn / etc.)
     │  - Forwards request to that instance
     │
     ▼
[ Backend Server Instance ]
     │  - Processes the request (hits DB, runs business logic, etc.)
     │  - Returns HTTP response
     │
     ▲ (response travels back up the same stack)
     │
[ Load Balancer ]  →  passes response back upstream
     │
[ API Gateway ]    →  may log, transform, or strip headers
     │
[ Reverse Proxy ]  →  compresses, chunks, re-encrypts to HTTPS
     │
Your Browser       →  renders the page
```

### Why Each Layer Exists

| Layer | Core Job | Key Benefit |
|---|---|---|
| DNS | Translate domain → IP | Humans use names; networks use IPs |
| Reverse Proxy | HTTPS termination, compression, streaming | Offloads TLS, reduces bandwidth, hides internals |
| API Gateway | Auth, rate limiting, routing, enrichment | Single security + policy enforcement point |
| Load Balancer | Distribute traffic, health check | Availability, horizontal scalability |
| Backend Server | Business logic | Actually answers the request |

Each layer has a **single clear responsibility**. Together, they form a pipeline that is secure, scalable, observable, and resilient to individual component failures.

---

> **Key Insight:** Notice that a significant amount of work happens *before* your application code ever runs. DNS resolution, TLS negotiation, authentication, rate limiting, routing, and load balancing all complete before a single line of your business logic executes. This separation of concerns is what allows each layer to be scaled, replaced, or upgraded independently.