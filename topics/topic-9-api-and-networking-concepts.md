# API Design & Networking Concepts
### A Plain-English Deep Dive

---

## Table of Contents

1. [What is API Design?](#1-what-is-api-design)
2. [REST, RPC, gRPC, and GraphQL](#2-rest-rpc-grpc-and-graphql)
3. [The Biggest Problems with REST](#3-the-biggest-problems-with-rest)
4. [When is gRPC a Better Choice?](#4-when-is-grpc-a-better-choice)
5. [HTTP Verbs in REST](#5-http-verbs-in-rest)
6. [What is TCPDump?](#6-what-is-tcpdump)
7. [HTTP vs HTTPS](#7-http-vs-https)
8. [The Math Behind Key Sharing: Diffie-Hellman](#8-the-math-behind-key-sharing-diffie-hellman)
9. [Pagination: Cursor vs Offset](#9-pagination-cursor-vs-offset)

---

## 1. What is API Design?

An **API (Application Programming Interface)** is a contract between two pieces of software. It defines *how* one system can ask another system for something — what to ask, how to ask it, and what to expect in return. Think of it like a restaurant menu: you don't go into the kitchen and cook yourself, you read the menu, place an order through a waiter (the API), and food comes out.

**API design** is the art and discipline of crafting that contract well. A well-designed API is intuitive, predictable, stable, and a joy to work with. A poorly-designed one is a source of bugs, confusion, and long nights.

### Best Practices for API Design

**1. Be consistent**

Pick a naming convention and stick with it everywhere. If one endpoint uses `camelCase` for field names and another uses `snake_case`, every consumer of your API will be confused. Consistency extends to URL structures, error formats, status codes, and date formats. Treat your API like a language — it needs grammar rules.

**2. Use nouns, not verbs, in URLs**

Your URLs should represent *resources* (things), not *actions* (verbs). The HTTP method already carries the action.

- ✅ `GET /users/42`
- ❌ `GET /getUser?id=42`

The verb `GET` tells the system what to do; `/users/42` tells it what thing you're acting on.

**3. Version your API from day one**

APIs change. If you don't version, every change is a breaking change for your users. Put a version in the URL or in a header right from the start.

- `/v1/users`
- `/v2/users`

This lets you evolve your API without breaking old clients.

**4. Use standard HTTP status codes correctly**

Don't return `200 OK` when something went wrong. Clients rely on status codes to understand the outcome of a request without parsing the body.

| Code | Meaning |
|------|---------|
| `200` | Success |
| `201` | Created |
| `400` | Bad request (client's fault) |
| `401` | Unauthenticated |
| `403` | Unauthorized (authenticated but not allowed) |
| `404` | Not found |
| `409` | Conflict (e.g. duplicate entry) |
| `422` | Unprocessable entity (validation failed) |
| `500` | Server error |

**5. Design error responses carefully**

Always return a consistent, human-readable error structure. Don't just send back `"error": true`. Give the developer something they can act on.

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "The 'email' field must be a valid email address.",
    "field": "email"
  }
}
```

**6. Make it idempotent where possible**

An **idempotent** operation is one you can call multiple times and always get the same result. `GET` is naturally idempotent. `PUT` (replace a resource) should also be idempotent. `POST` (create) typically is not. Designing for idempotency makes retries safe, which is critical in distributed systems.

**7. Secure by default**

Authentication and authorization should never be an afterthought. Use tokens (JWT, OAuth) over sessions. Use HTTPS everywhere. Apply the principle of least privilege — each API key or token should only have access to what it absolutely needs.

**8. Document everything**

An undocumented API is a trap. Use tools like OpenAPI/Swagger to auto-generate documentation from your code. A developer should be able to understand your API without reading your source code.

**9. Think about your consumers**

Your API is a product. Ask: who will use this? What are they trying to accomplish? Design around their use cases, not your internal data models. This is especially important for public APIs.

---

## 2. REST, RPC, gRPC, and GraphQL

These are four different **styles** or **protocols** for building APIs. They each answer the question "how should clients talk to servers?" in completely different ways.

---

### REST (Representational State Transfer)

REST is not a protocol or a standard — it's an **architectural style** described by Roy Fielding in his 2000 PhD dissertation. It was inspired by how the web itself works.

The core idea is simple: everything on the server is a **resource**, and you interact with those resources using standard HTTP methods (GET, POST, PUT, DELETE). Resources are identified by URLs, and the representation of a resource (usually JSON) is transferred back and forth.

A RESTful API looks like this:

```
GET    /articles          → list all articles
GET    /articles/5        → get article with id 5
POST   /articles          → create a new article
PUT    /articles/5        → replace article 5
PATCH  /articles/5        → partially update article 5
DELETE /articles/5        → delete article 5
```

REST is **stateless**, meaning each request must contain all the information needed to process it. The server doesn't remember previous requests.

**Strengths:** Simple to understand, leverages HTTP natively, excellent tooling and caching support, widely adopted.

**Weaknesses:** Can lead to over-fetching (getting more data than you need) or under-fetching (needing multiple requests to get all the data you want).

---

### RPC (Remote Procedure Call)

RPC takes a very different perspective. Instead of thinking in terms of resources, it thinks in terms of **actions**. You're not fetching a resource — you're calling a function on a remote machine.

The mental model is: "I want to call `sendEmail(to, subject, body)` on the server, and I don't particularly care that it maps to some URL resource. I just want to call a function."

Traditional RPC systems (like XML-RPC or SOAP) were common in the early 2000s. They often used XML over HTTP or raw TCP sockets.

**Strengths:** Very natural for action-heavy APIs (like internal microservices). Maps directly to how code works — you call functions.

**Weaknesses:** Less discoverable than REST. Older implementations were verbose and complex (SOAP, for example, was notoriously painful to work with).

---

### gRPC (Google Remote Procedure Call)

gRPC is Google's modern take on RPC, open-sourced in 2016. It dramatically improves on classic RPC in several ways.

**How it works:**

1. You define your service and its methods in a `.proto` file using **Protocol Buffers** (protobuf), which is a language-neutral, binary serialization format.
2. You run the protobuf compiler, which generates client and server code in your language of choice (Go, Python, Java, etc.).
3. Under the hood, gRPC uses **HTTP/2**, which enables multiplexing, header compression, and bidirectional streaming.

A `.proto` definition looks like this:

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}

message GetUserRequest {
  int32 id = 1;
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

From this definition, gRPC generates strongly-typed client stubs. You call `userClient.GetUser({id: 42})` in your code and the framework handles serialization, network transport, and deserialization.

**Strengths:** Extremely fast (binary, not JSON). Strongly typed contracts. Native streaming support. Great for internal microservice communication. Auto-generated client code.

**Weaknesses:** Not human-readable (binary wire format). Harder to debug with simple tools like `curl`. Not great for browser-based clients natively (requires gRPC-Web adapter).

---

### GraphQL

GraphQL is a **query language for APIs**, developed by Facebook (Meta) and open-sourced in 2015. It takes a fundamentally different approach from REST: instead of multiple endpoints, there is a **single endpoint**, and the client specifies exactly what data it wants.

Think of it this way: with REST, the server decides the shape of the data. With GraphQL, **the client decides**.

A GraphQL query might look like this:

```graphql
query {
  user(id: "42") {
    name
    email
    posts {
      title
      publishedAt
    }
  }
}
```

This returns *exactly* the fields you asked for — no more, no less. If you don't ask for `profilePicture`, you won't get it.

GraphQL uses three operation types:
- **Query** — read data
- **Mutation** — write/change data
- **Subscription** — real-time data pushed to the client

**Strengths:** Eliminates over-fetching and under-fetching. Single endpoint. Self-documenting via introspection. Excellent for complex, nested data requirements (like a social feed). Client-driven development.

**Weaknesses:** Caching is harder (everything is a POST to one URL). N+1 query problems on the backend if not careful (requires DataLoader or similar). More complex to implement server-side than REST.

---

## 3. The Biggest Problems with REST

REST is simple, but that simplicity comes with real costs at scale. Here are the most significant pain points.

---

### The JSON Payload Problem

This is the big one. REST APIs almost universally use **JSON** (JavaScript Object Notation) for data transfer. JSON is human-readable, which is great for debugging, but it comes with serious performance and efficiency costs.

**JSON is a text format.** Every number, boolean, date, and string is serialized as plain text characters. Compare these two representations of the same data:

- A 32-bit integer `12345` in binary takes **4 bytes**.
- The same number as a JSON string `"12345"` takes **5 bytes** — and that's if it's just a bare number with no field name attached. In practice you'd have `"userId": 12345` which is **16 bytes**.

Now scale that to millions of requests per second. The difference adds up to enormous bandwidth costs and slower serialization/deserialization.

**JSON parsing is expensive.** When a server receives a JSON payload, it must:
1. Read the raw bytes
2. Decode the string
3. Parse the JSON syntax (finding `{`, `}`, `:`, `,`)
4. Allocate objects in memory for every key and value
5. Map those to the internal data types

This takes CPU time. For high-throughput systems (think 100,000 requests/second), JSON parsing becomes a measurable bottleneck.

**JSON has no schema by default.** There's no built-in way to say "this field must be an integer" or "this field is required." You need external tooling (JSON Schema, Joi, Zod, etc.) to add validation. Without it, a typo like `"userID"` vs `"userId"` causes silent bugs.

**JSON has type ambiguity.** JSON has no native type for dates. You might receive `"2024-01-15"`, `"2024-01-15T00:00:00Z"`, or `1705276800000` — all valid, all requiring different parsing logic. Similarly, JSON doesn't distinguish between a 32-bit and 64-bit integer, which causes real problems when dealing with large IDs in JavaScript (which can't safely represent integers larger than 2^53).

---

### Over-fetching and Under-fetching

With REST, the server defines the shape of the response. This leads to two classic problems:

**Over-fetching** means receiving more data than you need. A `GET /users/42` endpoint might return the user's name, email, bio, profile picture URL, join date, and follower count — but you only needed the name for a dropdown. You've transmitted and parsed all that extra data for nothing.

**Under-fetching** (also called the N+1 problem) means one request isn't enough. You call `GET /posts` and get 20 posts, but each post just has an `authorId`. To display the author name, you now need to make 20 more `GET /users/{id}` calls. One request became 21.

---

### No Native Streaming or Bidirectionality

REST over HTTP/1.1 is fundamentally a request/response model. The client asks, the server answers, the connection closes (or is reused but still follows request/response cycles). There's no native way for the server to push updates to the client or to stream data bidirectionally. You need workarounds like long-polling, Server-Sent Events, or WebSockets — all of which are separate mechanisms bolted on top.

---

### Versioning Headaches

As your API evolves, you need to avoid breaking existing clients. REST has no built-in versioning mechanism. You typically end up with `/v1/`, `/v2/`, etc. in your URLs, which means maintaining multiple versions of your codebase indefinitely.

---

## 4. When is gRPC a Better Choice?

gRPC shines in specific situations. It is not universally better than REST — it's a different tool for different problems.

---

### Internal Microservice Communication

When you have dozens of services talking to each other inside your own infrastructure, gRPC is often the right call. The services are under your control, so you don't need human-readable JSON for debugging convenience. What you *do* need is speed, type safety, and a clear contract between services.

With gRPC, your `.proto` files serve as the single source of truth for how services communicate. If a service changes its API in a breaking way, the generated code won't compile — you catch the error at build time, not in production at 3am.

---

### High-Throughput, Low-Latency Systems

Because gRPC uses Protocol Buffers (binary) over HTTP/2, it is substantially faster than REST+JSON. Benchmarks routinely show gRPC being 5-10x faster than equivalent REST calls. For systems where latency and throughput actually matter — financial trading systems, real-time game backends, telemetry pipelines — this matters enormously.

---

### Streaming Data

gRPC has four communication patterns built in:

1. **Unary** — classic request/response (like REST)
2. **Server streaming** — client sends one request, server streams many responses (e.g., subscribing to live price updates)
3. **Client streaming** — client streams many messages, server replies once (e.g., uploading a large dataset chunk by chunk)
4. **Bidirectional streaming** — both sides stream simultaneously (e.g., a live chat or collaborative editing session)

REST has no native answer to streaming scenarios. gRPC handles them elegantly.

---

### Polyglot Environments

If you have services written in Go, Python, Java, and Kotlin all needing to talk to each other, gRPC is a great equalizer. You define the contract once in a `.proto` file and generate clients for every language. The contract is the same regardless of implementation language.

---

### When NOT to Use gRPC

- **Public-facing APIs**: Browsers can't natively use gRPC (you need gRPC-Web + a proxy). Developers expect REST+JSON for public APIs.
- **Simple CRUD APIs**: The overhead of setting up protobuf definitions isn't worth it for a simple blog backend.
- **When debuggability is paramount**: You can't just `curl` a gRPC endpoint and read the response. The binary format makes ad-hoc inspection harder.

---

## 5. HTTP Verbs in REST

HTTP defines a set of **methods** (often called "verbs") that describe the intended action on a resource. REST uses these methods to express CRUD operations (Create, Read, Update, Delete).

---

### GET

**Purpose:** Retrieve a resource. Read-only. Should never have side effects.

**Idempotent:** Yes. Calling `GET /articles/5` ten times should always return the same article (assuming no one changed it).

**Safe:** Yes. "Safe" means the operation doesn't modify anything on the server.

```
GET /articles        → returns a list of all articles
GET /articles/5      → returns the article with id 5
```

---

### POST

**Purpose:** Create a new resource, or trigger a process. The most general-purpose verb.

**Idempotent:** No. Calling `POST /articles` twice will (should) create two articles.

**Safe:** No.

```
POST /articles       → creates a new article from the request body
POST /payments       → processes a payment (not a pure resource creation)
```

---

### PUT

**Purpose:** Replace an entire resource with the one provided. If the resource doesn't exist, some implementations will create it (upsert).

**Idempotent:** Yes. Calling `PUT /articles/5` with the same body ten times should always result in the same article — you're just replacing it with the same thing.

**Safe:** No.

```
PUT /articles/5      → replace article 5 entirely with the new content
```

The key word is *replace*. If your article object has 10 fields and you PUT with only 3 fields, the other 7 should be cleared/null.

---

### PATCH

**Purpose:** Partially update a resource. Only the fields you send are changed; the rest remain untouched.

**Idempotent:** Usually yes, but not guaranteed. `PATCH /counter {increment: 1}` would not be idempotent.

**Safe:** No.

```
PATCH /articles/5    → update only the title of article 5
```

Use PATCH when you want surgical updates. Use PUT when you're replacing the whole thing.

---

### DELETE

**Purpose:** Remove a resource.

**Idempotent:** Yes. Deleting something that's already gone should return either `200 OK` or `404 Not Found` — but calling it multiple times shouldn't cause an error cascade.

**Safe:** No.

```
DELETE /articles/5   → delete article 5
```

---

### HEAD

**Purpose:** Same as GET, but the server returns only the headers, not the body. Useful for checking if a resource exists or checking its metadata (like `Content-Length` or `Last-Modified`) without downloading it.

**Idempotent:** Yes. **Safe:** Yes.

---

### OPTIONS

**Purpose:** Returns the HTTP methods supported by a resource. Used by browsers to send a "preflight" request in CORS (Cross-Origin Resource Sharing) to check if the actual request is allowed.

```
OPTIONS /articles    → returns: Allow: GET, POST, HEAD, OPTIONS
```

---

### Quick Reference Table

| Method | CRUD  | Idempotent | Safe | Has Body |
|--------|-------|-----------|------|----------|
| GET    | Read  | ✅ | ✅ | No |
| POST   | Create | ❌ | ❌ | Yes |
| PUT    | Update (replace) | ✅ | ❌ | Yes |
| PATCH  | Update (partial) | Usually ✅ | ❌ | Yes |
| DELETE | Delete | ✅ | ❌ | Sometimes |
| HEAD   | Read (headers only) | ✅ | ✅ | No |
| OPTIONS | Read (capabilities) | ✅ | ✅ | No |

---

## 6. What is TCPDump?

**tcpdump** is a command-line network packet analyzer. It runs on your machine, hooks into a network interface, and captures all the raw network traffic flowing through it. Think of it as a wiretap for your computer's network connection.

### What does it actually do?

Every time your computer sends or receives data — an HTTP request, a DNS lookup, a database query — that data travels as a series of **packets** across the network. tcpdump intercepts those packets at the kernel level and lets you inspect them.

You can filter by:
- Source or destination IP address
- Port number (e.g., port 80 for HTTP, port 443 for HTTPS)
- Protocol (TCP, UDP, ICMP)
- Specific hosts or networks

### Basic Usage

```bash
# Capture all traffic on the default network interface
sudo tcpdump

# Capture only HTTP traffic (port 80)
sudo tcpdump port 80

# Capture traffic to/from a specific host
sudo tcpdump host 192.168.1.100

# Capture and save to a file for later analysis
sudo tcpdump -w capture.pcap

# Read from a saved capture file
sudo tcpdump -r capture.pcap

# Show verbose output and don't resolve hostnames (faster)
sudo tcpdump -v -n

# Capture traffic on a specific interface
sudo tcpdump -i eth0
```

### Why is tcpdump useful?

**Debugging network issues:** If your application can't connect to a database, tcpdump tells you whether packets are even being sent, whether they're reaching the destination, and whether responses are coming back. It takes the guesswork out of "is this a code problem or a network problem?"

**Security analysis:** You can see exactly what data is being transmitted. This is how you verify that sensitive data is actually encrypted, or detect unexpected outbound connections from a compromised machine.

**Performance analysis:** By looking at the timing between packets, you can identify network latency issues, TCP retransmissions (which indicate packet loss), and slow server responses.

**Understanding protocols:** tcpdump is one of the best ways to learn how protocols actually work — you can watch a TCP handshake happen in real time, see DNS queries and responses, and observe HTTP headers in the clear.

### tcpdump vs Wireshark

Wireshark does the same thing but with a graphical interface. They use the same underlying library (libpcap). tcpdump is preferred on servers where you don't have a GUI, or when you want to capture traffic and analyze it later. A common workflow is to capture with tcpdump on a server, transfer the `.pcap` file, and open it in Wireshark for visual analysis.

### Important note

tcpdump requires elevated privileges (root/sudo) because intercepting network traffic is a powerful capability. It should only be used on networks and systems you have explicit permission to analyze.

---

## 7. HTTP vs HTTPS

### HTTP (HyperText Transfer Protocol)

HTTP is the foundation of data communication on the web. When your browser asks for a webpage, it sends an HTTP request, and the server responds with an HTTP response. The request and response look something like this:

**Request:**
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
```

**Response:**
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...
```

The problem with HTTP is that everything is sent in **plain text**. If you're sitting on the same Wi-Fi network as someone using HTTP, you can run tcpdump and read every word of their traffic. You'd see their login credentials, their messages, the pages they're visiting — everything. This is called a **Man-in-the-Middle (MitM) attack**.

### HTTPS (HTTP Secure)

HTTPS is HTTP with a security layer on top. It wraps the HTTP communication in **TLS (Transport Layer Security)**, which was previously called SSL. The "S" in HTTPS stands for "Secure."

TLS provides three guarantees:

1. **Confidentiality** — The data is encrypted. Even if someone intercepts your packets, they see gibberish.
2. **Integrity** — The data cannot be tampered with in transit without detection. If a packet is modified, the TLS layer detects it and the connection is rejected.
3. **Authentication** — You can verify you're actually talking to the server you think you are, not an impersonator. This is done via **certificates** issued by trusted Certificate Authorities (CAs).

### The TLS Handshake

Before any encrypted data is exchanged, the client and server do a **TLS handshake** to establish a shared secret (a session key). At a high level:

1. **Client Hello** — The client says "I'd like to connect securely" and lists the encryption algorithms it supports.
2. **Server Hello** — The server picks an algorithm and sends its **certificate** (which contains its public key).
3. **Certificate Verification** — The client checks the certificate against a list of trusted CAs to verify the server is who it claims to be.
4. **Key Exchange** — The client and server use their certificates and some math (see Diffie-Hellman below) to agree on a shared session key without ever transmitting that key directly.
5. **Encrypted Communication Begins** — Everything from here is encrypted with the shared session key.

### The Performance Trade-off

HTTPS is slightly slower than HTTP due to:
- The TLS handshake adds one or two extra round trips when establishing a new connection
- Encryption and decryption take CPU time

However, with modern hardware and HTTP/2 (which HTTPS enables), these costs are negligible. There's essentially no reason to use plain HTTP for any production website or API today.

### Visual Comparison

| | HTTP | HTTPS |
|-|------|-------|
| Data format | Plain text | Encrypted |
| Default port | 80 | 443 |
| Interceptable? | Yes | No |
| Tamper-detectable? | No | Yes |
| Server authentication | No | Yes (via certificates) |
| Performance | Marginally faster | Effectively the same with TLS 1.3 |
| Suitable for production? | No | Yes |

---

## 8. The Math Behind Key Sharing: Diffie-Hellman

This is one of the most elegant ideas in modern cryptography. The problem it solves is: **how can two people agree on a secret key over a public channel, where anyone can listen in, and yet no eavesdropper can figure out the key?**

Intuitively this seems impossible. If Alice and Bob are shouting across a room full of people, how can they agree on a secret that nobody else hears?

Diffie-Hellman, invented by Whitfield Diffie and Martin Hellman in 1976, solves this beautifully.

---

### The Paint Analogy (Intuition First)

Before the math, here's a physical analogy that captures the essence:

1. Alice and Bob publicly agree on a common paint color — let's say **yellow**. Everyone can see this.
2. Alice secretly picks her own color — **orange** — and mixes it with the yellow to get **yellow-orange**. She sends this to Bob.
3. Bob secretly picks his own color — **blue** — and mixes it with the yellow to get **yellow-blue**. He sends this to Alice.
4. Alice takes Bob's **yellow-blue** and adds her secret **orange** → she gets **yellow-orange-blue**.
5. Bob takes Alice's **yellow-orange** and adds his secret **blue** → he also gets **yellow-orange-blue**.

They've both arrived at the same color! An eavesdropper only saw yellow, yellow-orange, and yellow-blue. Without knowing Alice's secret orange or Bob's secret blue, they can't recreate **yellow-orange-blue**.

The key insight: **mixing paint is easy, but un-mixing paint is hard.** This is the one-way function at the heart of Diffie-Hellman.

---

### The Actual Math

The real protocol replaces "paint mixing" with **modular arithmetic** and the **discrete logarithm problem**.

**Setup (public parameters):**
- A large prime number `p` (publicly known, usually thousands of bits long)
- A generator `g` (a number between 1 and p, also public)

**The Exchange:**

1. Alice picks a secret number `a` (private, never shared)
2. Bob picks a secret number `b` (private, never shared)

3. Alice computes her public value:
   ```
   A = g^a mod p
   ```
   She sends `A` to Bob.

4. Bob computes his public value:
   ```
   B = g^b mod p
   ```
   He sends `B` to Alice.

5. Alice computes the shared secret:
   ```
   s = B^a mod p = (g^b)^a mod p = g^(ab) mod p
   ```

6. Bob computes the shared secret:
   ```
   s = A^b mod p = (g^a)^b mod p = g^(ab) mod p
   ```

**They both arrive at the same secret `s = g^(ab) mod p`.**

---

### Why Can't an Eavesdropper Crack It?

An eavesdropper knows `p`, `g`, `A`, and `B` (all transmitted publicly). To find the shared secret, they would need to figure out `a` from `A = g^a mod p`, or `b` from `B = g^b mod p`.

This is the **discrete logarithm problem**: given `g`, `p`, and `A`, find `a`. For large primes (2048+ bits), this is computationally infeasible with current technology. The best known algorithms take longer than the age of the universe to run on modern hardware.

---

### A Tiny Concrete Example

Let's use small numbers (real implementations use numbers with thousands of digits).

- Public: `p = 23`, `g = 5`
- Alice's secret: `a = 6`
- Bob's secret: `b = 15`

Alice sends: `A = 5^6 mod 23 = 15625 mod 23 = 8`

Bob sends: `B = 5^15 mod 23 = 30517578125 mod 23 = 19`

Alice computes: `s = 19^6 mod 23 = 47045881 mod 23 = 2`

Bob computes: `s = 8^15 mod 23 = 35184372088832 mod 23 = 2`

**Both get 2.** ✅

The eavesdropper knows `g=5`, `p=23`, `A=8`, `B=19`. They need to find `a` (the number such that `5^a mod 23 = 8`). With small numbers they could brute-force it, but with 2048-bit primes, it's impossible.

---

### How This Connects to HTTPS

During a TLS handshake, the client and server use a variant called **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** to agree on a session key. This session key is then used to encrypt all traffic. The word "Ephemeral" is important — a new key is generated for every session, which means that even if a long-term private key is compromised in the future, past sessions can't be decrypted. This property is called **Perfect Forward Secrecy (PFS)**.

---

## 9. Pagination: Cursor vs Offset

When you have a database table with millions of rows, you can't return everything at once. The client would time out, the server would run out of memory, and the user would be staring at a loading screen forever. **Pagination** is the mechanism for returning data in manageable chunks.

There are two main strategies, and they are not equal.

---

### Offset/Page-Based Pagination

This is the intuitive, obvious approach. You divide your data into pages of size N, and the client asks for page 1, page 2, page 3, etc.

**How it works:**

```sql
-- Get page 3, with 20 items per page
SELECT * FROM articles ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

The `OFFSET 40` tells the database to skip the first 40 rows and return the next 20.

In API terms:
```
GET /articles?page=3&per_page=20
GET /articles?limit=20&offset=40
```

**Why it feels natural:** It maps perfectly to how humans think about books and pages. "I'm on page 5" is easy to understand and allows jumping to arbitrary pages.

---

### The Problems with Offset Pagination

**Problem 1: The database still scans all skipped rows.**

`OFFSET 40` doesn't mean "start at row 41." It means "scan all 41 rows, then throw away the first 40." For page 500 of 20 items, the database scans 10,000 rows to throw them away. Performance degrades linearly with how deep into the dataset you paginate. Page 1 is fast. Page 1,000,000 is catastrophically slow.

**Problem 2: Data shifting causes duplicates and skipped items.**

Imagine a user is reading through a feed. They load page 1 (items 1-20). While they're reading, 5 new items are inserted at the top. They ask for page 2 (offset 20). The database now counts from the new total, so the last 5 items from the old page 1 are now pushed down into page 2. The user sees them again — **duplicates**.

The reverse happens with deletions. If 5 items are deleted between page loads, the user skips over items that they never saw — **silent data loss**.

In rapidly-changing datasets like social media feeds, this is not a theoretical edge case. It happens constantly.

**Problem 3: Page numbers are meaningless in dynamic data.**

"Page 5" has no stable meaning. If the dataset changes, page 5 now shows completely different data. There's no reliable way to bookmark a position or resume from where you left off.

---

### Cursor-Based Pagination

Cursor-based pagination solves all of these problems. Instead of "give me page N," the client says "give me the next 20 items after this specific item."

**How it works:**

A **cursor** is an opaque token that encodes a position in the dataset. Typically it's a Base64-encoded representation of the last item's unique identifier or sort key (e.g., the timestamp of the last item seen).

**First request:**
```
GET /articles?limit=20
```

**Response:**
```json
{
  "data": [...20 articles...],
  "pagination": {
    "nextCursor": "eyJpZCI6IDIwfQ==",
    "hasMore": true
  }
}
```

**Next request:**
```
GET /articles?limit=20&cursor=eyJpZCI6IDIwfQ==
```

The server decodes the cursor, finds the position, and returns the next 20 items after that point:

```sql
SELECT * FROM articles
WHERE id < 20  -- (decoded from cursor)
ORDER BY id DESC
LIMIT 20;
```

This query can be answered instantly with an index, regardless of how deep into the dataset you are.

---

### Why Cursors Win

**1. No offset scanning.** The database uses the cursor value as a WHERE clause. It doesn't scan skipped rows — it jumps directly to the right position using an index. Page 1 and page 1,000,000 have the same query performance.

**2. Stable across data changes.** The cursor encodes "after item X," not "skip N rows." New items inserted at the top don't affect your position. Deleted items don't cause items to shift. You always get the next items after your last seen item.

**3. No duplicate items.** Because you're anchored to a specific item, not a count, shifting doesn't affect you.

---

### The Trade-offs of Cursor Pagination

Cursors aren't a free lunch. They come with real limitations:

- **No random access.** You can't jump to "page 50." You can only move forward (and sometimes backward). This is why Google Search uses cursor pagination (you go to next page) but never lets you jump to page 847.
- **Cursors can expire.** If items are deleted or the sort order changes, a cursor might become invalid.
- **More complex to implement.** The server must encode and decode cursors correctly. Off-by-one errors in cursor logic cause frustrating bugs.
- **Not intuitive for users.** "Previous / Next" is fine, but "Go to page 12" is impossible.

---

### When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Social feed, notifications, activity log | ✅ Cursor |
| Admin panel with "Go to page N" | ✅ Offset (static data) |
| Real-time data that changes frequently | ✅ Cursor |
| Large datasets (millions of rows) | ✅ Cursor |
| Small, relatively static dataset | Either works |
| Need to know total count / "X of Y pages" | Offset (with caveats) |

For any modern, high-traffic application with dynamic data, **cursor-based pagination is the default right answer.**

---

*End of document.*