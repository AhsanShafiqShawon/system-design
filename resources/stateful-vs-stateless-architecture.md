# Stateful vs Stateless Architecture, Sticky Sessions, and JWT vs Session Auth

> A plain-English deep dive into three foundational concepts every backend developer should understand.

---

## Table of Contents

1. [Stateful vs Stateless Architecture](#1-stateful-vs-stateless-architecture)
2. [Sticky Session Load Balancing](#2-sticky-session-load-balancing)
3. [JWT vs Session-Based Authentication](#3-jwt-vs-session-based-authentication)
4. [How It All Connects](#4-how-it-all-connects)

---

## 1. Stateful vs Stateless Architecture

### The Core Idea

Imagine you're a customer calling a bank's support line.

- **Stateful:** The same agent picks up every time. They *remember* who you are, your account history, and where you left off last time — no need to re-explain yourself.
- **Stateless:** A different agent picks up each time. They have no memory of past calls. Every time you call, you must bring your entire story with you — your name, account number, the issue — because they start from zero.

This is exactly the distinction in software.

---

### Stateful Architecture

In a **stateful** system, the **server remembers things about you between requests**. It stores your "state" — who you are, what you've done, what's in your cart, etc. — on the server side.

**How it works:**

1. User logs in.
2. Server creates a *session* and stores it in memory (or a database).
3. Server gives the user a small token (a Session ID cookie).
4. On the next request, the user sends that cookie.
5. The server looks up the session and says, *"Ah yes, this is Shawon, he's logged in, here's his cart."*

**The server is doing the remembering.**

**Real-world analogy:** A restaurant where the waiter remembers your usual order and your dietary restrictions without you having to say anything.

**Characteristics:**

| Property | Detail |
|---|---|
| Memory lives | On the **server** |
| Client sends | A small, meaningless ID |
| Server does | Looks up the full state using that ID |
| Problem | Every request *must* reach the **same server** that has your session |

---

### Stateless Architecture

In a **stateless** system, the server **remembers nothing**. Every single request the client sends must carry **all the information the server needs** to process it. The server reads the request, does its job, and forgets everything the moment it responds.

**How it works:**

1. User logs in.
2. Server creates a *token* (like a JWT) that contains the user's identity, roles, expiry time, etc.
3. Server sends the token to the client.
4. On every request, the client sends this token.
5. The server reads the token, verifies it cryptographically, and serves the request — **no database lookup needed**.

**The client is doing the remembering.**

**Real-world analogy:** A ticket at a theme park. The ticket itself contains all the information (valid dates, which rides, VIP access). The security guard doesn't need to call headquarters — they just read the ticket.

**Characteristics:**

| Property | Detail |
|---|---|
| Memory lives | On the **client** (in the token) |
| Client sends | A self-contained token with all needed info |
| Server does | Verifies the token's signature, reads its contents |
| Advantage | Any server can handle any request — no shared state needed |

---

### Side-by-Side Comparison

| | Stateful | Stateless |
|---|---|---|
| **Where state lives** | Server | Client (token) |
| **Server memory usage** | High (stores sessions) | Low (nothing to store) |
| **Scalability** | Harder — servers must share state | Easier — any server can respond |
| **Security on logout** | Easy — just delete the session | Harder — token lives until expiry |
| **Typical mechanism** | Sessions + Cookies | JWT |
| **Example use case** | Traditional web apps | REST APIs, microservices |

---

### Why This Matters for Scaling

This is where things get really interesting. Imagine your app becomes popular and you need **three servers** instead of one.

With a **stateful** setup, a problem emerges: if Server A stored your session, but your next request lands on Server B — Server B has no idea who you are and logs you out. This is the core pain point.

With a **stateless** setup, this problem simply doesn't exist. Every server can verify your JWT on its own. No coordination needed.

---

## 2. Sticky Session Load Balancing

### The Problem It Solves

As described above, stateful apps break when multiple servers are involved because the session only lives on **one** server. Sticky sessions are a **workaround** for this problem.

---

### What Is a Sticky Session?

A **sticky session** (also called **session affinity**) is a load balancer configuration where **every request from a specific user is always routed to the same server**.

The user gets "stuck" to one server for the duration of their session. That server holds their session data, and because the load balancer ensures they always land there, it always works.

**Real-world analogy:** A busy hospital with multiple reception desks. Normally, you'd go to whoever is free. But with sticky sessions, you're assigned *your* receptionist for the whole visit. They know your paperwork, your history, your situation. You always go back to them.

---

### How It Works Technically

The load balancer uses a **cookie or IP address** to track which server a user belongs to.

**Cookie-based stickiness (most common):**

1. User's first request hits the load balancer.
2. Load balancer picks a server (e.g., Server 2) and forwards the request.
3. The load balancer sets a special cookie on the user's browser: `SERVERID=server-2`.
4. On every future request, the browser sends that cookie.
5. The load balancer reads it and always routes to Server 2.

**IP-based stickiness:**

The load balancer hashes the user's IP address to always map it to the same server. Simpler, but less reliable — users behind NAT or VPNs can appear to switch IPs.

---

### The Problems with Sticky Sessions

Sticky sessions solve the stateful scaling problem, but they introduce new ones:

**1. Uneven load distribution**

If a power user or a bot sticks to Server 1 and drives heavy traffic, Server 1 gets overloaded while Servers 2 and 3 sit idle. The load balancer can't rebalance because that user is *glued* to Server 1.

**2. Single point of failure**

If Server 1 goes down, **all users stuck to it lose their sessions** and get logged out. Other servers can't help them because they don't have the session data.

**3. Harder to scale down**

You can't easily remove Server 1 from the pool without kicking out all its users. You have to wait for sessions to expire or drain traffic first.

**4. Doesn't truly solve the problem**

Sticky sessions are a band-aid. They work around stateful architecture rather than fixing it. The real solution is either:
- Moving sessions to a **shared external store** (like Redis), so any server can read any session, OR
- Going **stateless** (JWT) so there's no session to share at all.

---

### Sticky Sessions vs Shared Session Store

| | Sticky Sessions | Shared Session Store (Redis) |
|---|---|---|
| **How it works** | Route same user to same server | All servers share one session database |
| **Fault tolerance** | Poor — server failure = lost session | Good — data survives server failures |
| **Load balancing quality** | Uneven | Even — any server handles any request |
| **Complexity** | Low — just a load balancer setting | Medium — need to run and manage Redis |
| **Best for** | Quick fix, legacy apps | Proper stateful distributed systems |

---

## 3. JWT vs Session-Based Authentication

This section brings the previous two topics together into the most practical decision you'll make in a web app: **how to handle auth**.

---

### Session-Based Authentication (The Classic Way)

**Step-by-step flow:**

```
1. User POSTs /login with username + password
2. Server verifies credentials against the database
3. Server creates a session:  { userId: 42, role: "admin", cartId: 99 }
4. Server stores this session in memory (or Redis, or DB)
5. Server generates a random Session ID:  "abc123xyz"
6. Server sends:  Set-Cookie: sessionId=abc123xyz; HttpOnly; Secure
7. Browser stores the cookie automatically
8. On every future request, browser sends:  Cookie: sessionId=abc123xyz
9. Server receives it, looks up "abc123xyz" in the session store
10. Finds the session data → knows who the user is → serves the request
```

**The session ID itself means nothing** — it's just a random string. All the real data is on the server. The client is just holding a "claim ticket."

**Logout:**
Server deletes the session from the store. The cookie becomes useless. Clean and instant.

---

### JWT — JSON Web Token (The Modern Way)

A JWT is a **self-contained token** that holds all the user's information inside it, cryptographically signed by the server.

**Anatomy of a JWT:**

A JWT looks like this:

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjQyLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3MTk...
```

It has **three parts**, separated by dots: `Header.Payload.Signature`

**1. Header** — describes the token type and signing algorithm:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2. Payload** — the actual data (called "claims"):
```json
{
  "userId": 42,
  "role": "admin",
  "email": "shawon@example.com",
  "exp": 1719875200
}
```

**3. Signature** — a cryptographic hash of the header + payload, signed with a secret key:
```
HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

The signature is what makes the token tamper-proof. If anyone changes the payload (e.g., tries to change `"role": "user"` to `"role": "admin"`), the signature won't match and the server will reject it.

---

**Step-by-step flow:**

```
1. User POSTs /login with username + password
2. Server verifies credentials
3. Server creates a JWT with the user's data + an expiry time
4. Server signs it with a secret key
5. Server sends the JWT to the client
6. Client stores it (localStorage, sessionStorage, or a cookie)
7. On every request, client sends:  Authorization: Bearer <token>
8. Server receives the JWT, verifies the signature
9. If valid and not expired → reads the payload → knows who the user is
10. No database lookup needed
```

**Logout:**
This is the tricky part. The server can't "delete" a JWT because it doesn't store it. You can only:
- Let it expire naturally (set short expiry times, e.g., 15 minutes)
- Maintain a "blacklist" of invalidated tokens (but then you're back to server-side state!)
- Use short-lived access tokens + longer-lived refresh tokens

---

### The Token Refresh Pattern

Because JWT logout is hard, most production apps use **two tokens**:

| Token | Lifetime | Stored | Purpose |
|---|---|---|---|
| **Access Token** | 15 minutes | Memory / header | Used on every API request |
| **Refresh Token** | 7–30 days | HttpOnly cookie | Used only to get a new access token |

**Flow:**
1. User logs in → gets both tokens.
2. Access token is used for API calls.
3. After 15 min, access token expires.
4. Client silently sends refresh token to `/refresh`.
5. Server validates refresh token (checks a DB), issues new access token.
6. User never notices.
7. On logout, server deletes the refresh token from DB → revocation achieved.

---

### JWT vs Session: Full Comparison

| | Session-Based | JWT |
|---|---|---|
| **State location** | Server (stateful) | Client (stateless) |
| **Server storage** | Requires session store (memory/Redis/DB) | None |
| **Scalability** | Needs shared store or sticky sessions | Any server can verify — scales easily |
| **Logout** | Simple — delete session | Hard — token lives until expiry |
| **Token size** | Tiny (just an ID string) | Larger (contains all claims) |
| **Revocation** | Instant | Difficult without a blacklist |
| **Cross-domain / CORS** | Cookies can be tricky cross-domain | Easy — just send in `Authorization` header |
| **Microservices** | Each service needs to call session store | Each service verifies token independently |
| **Security of data** | Data never leaves server | Payload is base64-encoded, **not encrypted** — don't put secrets in it |
| **Best for** | Traditional web apps, monoliths | REST APIs, SPAs, mobile apps, microservices |

---

### A Common Misconception: JWT is NOT Encrypted

The payload of a JWT is only **base64url-encoded**, not encrypted. Anyone can decode it and read the contents:

```bash
# Decode a JWT payload (anyone can do this):
echo "eyJ1c2VySWQiOjQyLCJyb2xlIjoiYWRtaW4ifQ==" | base64 -d
# Output: {"userId":42,"role":"admin"}
```

**Never put sensitive data** (passwords, credit card numbers, SSNs) in a JWT payload.

The **signature** ensures integrity (can't be tampered with), but not confidentiality (can be read by anyone). If you need confidentiality, use JWE (JSON Web Encryption) instead of JWT.

---

## 4. How It All Connects

These three concepts are deeply related. Here's how they fit together:

```
┌─────────────────────────────────────────────────────────────────┐
│                    STATEFUL ARCHITECTURE                        │
│                                                                 │
│  Uses Sessions → Server stores state → Scaling problem          │
│                                                                 │
│  Solutions:                                                     │
│  ├── Sticky Sessions   (band-aid: route same user, same server) │
│  └── Shared Store      (proper: Redis/DB stores sessions)       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   STATELESS ARCHITECTURE                        │
│                                                                 │
│  Uses JWT → Client stores state → Scales effortlessly           │
│                                                                 │
│  Trade-off:                                                     │
│  └── Logout / token revocation requires extra care              │
└─────────────────────────────────────────────────────────────────┘
```

**The bottom line:**

- If you're building a **traditional web app or monolith**, **sessions with a shared Redis store** are robust, easy to revoke, and well-understood.
- If you're building a **REST API, microservices platform, or mobile app**, **JWT** is the natural fit — stateless, portable, and no shared infrastructure required.
- **Sticky sessions** are a pragmatic shortcut but carry real operational risk. Treat them as a temporary solution, not a foundation.