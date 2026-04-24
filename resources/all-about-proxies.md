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

## 7. How They All Relate

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

---