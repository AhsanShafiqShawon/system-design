# 🌐 All About DNS

---

## What is DNS?

Imagine the internet is a giant city, and every website lives at a specific address — but that address is a string of numbers like `142.250.80.46`. Nobody wants to memorize that.

**DNS (Domain Name System)** is the internet's phone book. It translates human-friendly names like `google.com` into the numerical IP addresses that computers actually use to talk to each other.

> Without DNS, you'd have to type a number like `142.250.80.46` instead of `google.com` every time you wanted to visit a website. DNS makes the web usable.

---

## How Does a DNS Server Work?

When you type a URL into your browser, a small behind-the-scenes conversation happens — fast, but layered. Here's how it unfolds:

### The Cast of Characters

| Role | What It Does |
|---|---|
| **DNS Resolver** | Your first stop. Usually run by your ISP or a service like Google (8.8.8.8) or Cloudflare (1.1.1.1). |
| **Root Name Server** | The top of the DNS hierarchy. Knows where to find servers for each top-level domain (`.com`, `.org`, etc.). |
| **TLD Name Server** | Handles a specific top-level domain. The `.com` TLD server knows where `google.com`'s records live. |
| **Authoritative Name Server** | The final authority. Holds the actual DNS records for a domain and gives the definitive answer. |

### The Journey of a DNS Query

```
You type: google.com
    │
    ▼
[1] DNS Resolver  ──── "Do I have this cached?" ────► Yes? → Return IP immediately
    │ No?
    ▼
[2] Root Name Server  ──── "Who handles .com?" ────► Points to TLD server
    │
    ▼
[3] TLD Name Server (.com)  ──── "Who handles google.com?" ────► Points to Google's authoritative server
    │
    ▼
[4] Authoritative Name Server  ──── "What is google.com's IP?" ────► Returns: 142.250.80.46
    │
    ▼
[5] DNS Resolver returns the IP to your browser
    │
    ▼
Your browser connects to 142.250.80.46 → Google loads ✅
```

The whole process typically takes **less than 50 milliseconds** — faster than a blink.

---

## What is DNS Caching?

Doing that full 5-step lookup for *every* website visit would be slow and wasteful. That's where **caching** comes in.

Caching means **storing a copy of the answer** so you don't have to ask the same question twice.

### Where Caching Happens

1. **Your browser** — Chrome, Firefox, and others keep their own DNS cache.
2. **Your operating system** — Windows, macOS, and Linux each maintain a system-level DNS cache.
3. **Your router** — Your home router often caches DNS responses too.
4. **Your DNS resolver** — The resolver (e.g., Cloudflare's 1.1.1.1) caches results for many users at once, which massively speeds things up.

### Why Caching Matters

- ⚡ **Speed** — Cached lookups return instantly, skipping all the back-and-forth.
- 📉 **Less traffic** — Fewer requests hit the root and TLD servers, keeping the internet's core infrastructure from being overwhelmed.
- 🔄 **Stale risk** — Cached data can become outdated. If a website changes its IP, people with old cached records might temporarily reach the wrong place (or nowhere at all).

---

## What is DNS TTL?

**TTL stands for Time To Live.** It's a number, measured in seconds, that tells caches how long they're allowed to keep a DNS record before they must throw it away and fetch a fresh copy.

Think of it like an expiry date on a carton of milk.

### How TTL Works

A domain owner sets the TTL in their DNS records. For example:

```
google.com    A record    142.250.80.46    TTL: 300
```

This means: *"You can cache this answer for 300 seconds (5 minutes). After that, ask again."*

### Choosing the Right TTL

| TTL Value | Suited For |
|---|---|
| **60–300 seconds** (short) | When you expect to change your IP soon, or during migrations. Changes propagate quickly, but more DNS queries are generated. |
| **3600 seconds** (medium, ~1 hour) | A balanced default for most websites. |
| **86400 seconds** (long, ~1 day) | Stable infrastructure that rarely changes. Faster for visitors, but slow to update if something changes. |

> **Pro tip:** If you're planning to move a website to a new server, lower your TTL to 300 seconds a day or two *before* the move. Once the migration is done and confirmed, raise it back up.

---

## What is DNS Propagation?

**DNS propagation** is the time it takes for a DNS change to spread across all the DNS servers around the world.

When you update a DNS record (like pointing your domain to a new server), that change doesn't happen everywhere at once. It ripples outward gradually — server by server, cache by cache — until the whole internet has the updated information.

### Why Doesn't It Happen Instantly?

Because of TTL. Every DNS server that has cached your old record will keep using it until that cache expires. Only then will it fetch the updated record.

If your TTL was set to 86400 seconds (24 hours) before you made a change, some users might see the old record for up to **24 hours** after your update.

### The Propagation Timeline

```
You update DNS record
        │
        ▼
Your authoritative server has the new record immediately ✅
        │
        ▼
Resolvers around the world still serve cached old record ⏳
        │
        ▼
One by one, as TTLs expire, resolvers fetch the new record ✅
        │
        ▼
Propagation complete — everyone sees the new record 🌍
```

Propagation typically takes **anywhere from a few minutes to 48 hours**, depending on the TTL that was set *before* the change was made.

### Common Propagation Scenarios

- **Launching a new website** — You've just pointed your domain to a new hosting provider. Some visitors may temporarily see "site not found" while propagation completes.
- **Moving to a new server** — Old records might still direct some traffic to your old server for a while.
- **Changing email records (MX records)** — Emails might still arrive at the old mail server until propagation finishes.

> You can check propagation progress using tools like [dnschecker.org](https://dnschecker.org), which shows whether DNS servers in different countries have picked up your changes yet.

---

## Best Practices: Lowering TTL Before a DNS Change

Changing DNS while your TTL is still high is like announcing a house move the moment you're already gone — people will keep showing up at the old address for a long time. The trick is to **prepare well in advance**.

Here's the recommended playbook for a smooth, near-zero-downtime DNS migration.

### The Step-by-Step Rundown

**Step 1 — Check your current TTL (well in advance)**

Before anything else, find out what your current TTL is. You can check it from the terminal:

```bash
# macOS / Linux
dig google.com

# Windows
nslookup -type=A google.com
```

Look for the TTL value in the response. If it's 86400 (24 hours) or higher, you need to start planning at least two days ahead.

**Step 2 — Lower your TTL (24–48 hours before the change)**

Log into your DNS provider and drop the TTL on the records you're planning to change down to **300 seconds (5 minutes)**. Don't make any other changes yet — just the TTL.

```
Before:   mysite.com    A    203.0.113.10    TTL: 86400
After:    mysite.com    A    203.0.113.10    TTL: 300
```

Now wait. You need to wait for *at least as long as your old TTL* — because caches around the world are still holding the old record with the old (long) TTL. Until those caches expire, the new short TTL hasn't taken effect globally.

> ⏳ **The waiting is the hardest part.** If you skip it, you'll make your DNS change while the world still has a 24-hour cache. Lowering TTL retroactively doesn't help caches that already have the old value locked in.

**Step 3 — Make the actual DNS change**

Once the wait is over, update the record to point to your new server:

```
mysite.com    A    198.51.100.55    TTL: 300
```

Because your TTL is now only 300 seconds, caches around the world will pick up the new IP within **5 minutes** of expiry — not 24 hours.

**Step 4 — Verify propagation**

Use a tool like [dnschecker.org](https://dnschecker.org) to confirm that resolvers across different regions are returning your new IP. Within a few minutes of making the change, you should see green ticks appearing globally.

**Step 5 — Raise your TTL back up**

Once the new server is confirmed healthy and propagation is complete, raise your TTL back to a sensible long-term value:

```
mysite.com    A    198.51.100.55    TTL: 3600
```

Leaving TTL at 300 indefinitely means every visitor triggers more frequent DNS lookups, adding small amounts of latency and putting more load on your authoritative name server.

---

### The Full Timeline at a Glance

```
T - 48h   Lower TTL from 86400 → 300 on your DNS provider
          │
T - 48h   Wait for old caches to expire (at least one full old-TTL cycle)
  to       │
T - 0h    World now fetches fresh records every 5 minutes ✅
          │
T = 0h    Make your actual DNS change (point to new server)
          │
T + 5min  Most resolvers have the new record ✅
          │
T + 1h    Confirm new server is healthy, propagation complete
          │
T + 1h    Raise TTL back up to 3600 (or your preferred value)
```

---

### Common Mistakes to Avoid

| Mistake | Why It Hurts | What to Do Instead |
|---|---|---|
| Lowering TTL and changing DNS at the same time | Caches still hold the old record with the old long TTL — you get no benefit from the lower TTL. | Lower TTL first, wait a full TTL cycle, *then* change. |
| Forgetting to raise TTL afterwards | Unnecessary DNS queries on every visit, slightly higher latency for users. | Set a calendar reminder to raise it after the migration. |
| Only lowering TTL on the A record | If you're also changing MX or CNAME records, those need the same treatment. | Lower TTL on *all* records you plan to change. |
| Checking propagation too early | Propagation takes time — panicking and making extra changes causes chaos. | Be patient. Use dnschecker.org and wait 5–10 minutes. |

---

## Quick Recap

| Concept | Plain English Summary |
|---|---|
| **DNS** | The internet's phone book — translates domain names to IP addresses. |
| **DNS Server** | A system of layered servers that work together to resolve a domain name. |
| **DNS Caching** | Storing a DNS answer locally so you don't have to look it up again. |
| **TTL** | An expiry timer on a cached DNS record — controls how long it's trusted before being refreshed. |
| **DNS Propagation** | The gradual spread of a DNS change across all servers worldwide, constrained by TTL. |

---