# Rate Limiter

## TL;DR

**A rate limiter controls how many requests a client can make in a given amount of time.**

"Max 100 requests per minute per user." Request 101 in that minute gets rejected (usually with HTTP **429 Too Many Requests**) instead of reaching your servers.

It protects your system from abuse, keeps one noisy client from ruining things for everyone, and keeps costs under control.

---

## The analogy that makes it click

**A rate limiter is the bouncer at a club with a capacity rule.**

The bouncer lets people in at a steady pace. If a bus of 200 people shows up at once, they don't all get in — some are told "wait and try again later." Nobody inside the club gets crushed, and the bar staff can keep up.

The rules can vary: "max 50 per hour", "each person gets a few entry tokens", "one group every 10 seconds." Those different rules are the different **algorithms** below.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Limit** | Max number of requests allowed (e.g. 100) |
| **Window** | The time period the limit applies to (e.g. per minute) |
| **Key** | *Who* is being limited — user ID, API key, IP address, or endpoint |
| **Throttling** | Slowing down or rejecting requests over the limit |
| **Burst** | A sudden short spike of requests |
| **429** | HTTP status code "Too Many Requests" |

---

## Why rate limit?

| Reason | Example |
|---|---|
| **Prevent abuse & attacks** | Stop brute-force password guessing, scraping, spam, and some DoS traffic |
| **Fairness** | One customer's buggy script sending 10k req/s shouldn't slow the API for everyone else |
| **Protect downstream systems** | Your DB can handle 5,000 queries/s — don't let traffic push it past that |
| **Control cost** | Each request may call a paid API (SMS, payment, AI model). Limits cap your bill. |
| **Enforce business plans** | Free tier: 1,000 calls/day. Pro tier: 100,000 calls/day. |

---

## Where it sits

```
Client ──► [ CDN / WAF ] ──► [ API Gateway / LB ] ──► [ App servers ] ──► DB
              ▲                    ▲                        ▲
         IP-based limits     per-user / per-API-key    fine-grained, per-feature
         (block floods)      limits (most common)       (e.g. 5 password resets/hour)
```

| Location | Pros | Cons |
|---|---|---|
| **Client side** | Reduces wasted calls | ❌ Can't be trusted — anyone can bypass it |
| **Edge (CDN/WAF)** | Stops attacks before they reach you | Coarse — usually only knows the IP |
| **API gateway / middleware** | Central, knows the user/API key, most common choice | Needs shared state across gateway instances |
| **Inside the service** | Can apply business-specific rules | Every service re-implements it |

**Always enforce on the server side.** Client-side limits are only a courtesy.

---

## The algorithms

### 1. Token bucket

A bucket holds up to **N tokens**. Tokens are added at a fixed rate (e.g. 10 per second). Each request takes one token. No token → request rejected.

```
   refill: +10 tokens/sec
          │
          ▼
     ┌─────────┐
     │ ● ● ● ● │  capacity = 20 tokens
     │ ● ● ● ● │
     └────┬────┘
          │ each request takes 1 token
          ▼
  request ──► token available? ── yes ──► allowed ✅
                               └─ no  ──► rejected (429) ❌
```

- A full bucket allows a **burst** of up to 20 requests at once, then settles to 10/s.
- Very common: used by AWS API Gateway, Stripe, and many others.
- Just two numbers to store per key: token count and last refill time.

### 2. Leaky bucket

Requests enter a queue (the bucket) and are processed at a **constant rate**, like water dripping from a hole. If the bucket is full, new requests are dropped.

```
  requests pour in (bursty)
      ▼ ▼▼▼ ▼
   ┌──────────┐
   │ ▓▓▓▓▓▓▓▓ │  queue of size N  (full? → drop new requests ❌)
   │ ▓▓▓▓▓▓▓▓ │
   └────┬─────┘
        │  leaks out at a FIXED rate (e.g. 10/sec)
        ▼
     processed ─ ─ ─ ─ ─   (smooth, steady output)
```

- Output is perfectly smooth — great for protecting a system that needs a steady load.
- Bursts are **not** allowed through faster; they wait or get dropped. Old queued requests can delay newer ones.

### 3. Fixed window counter

Split time into fixed windows (e.g. each minute). Keep a counter per window. Reject when the counter exceeds the limit.

```
limit = 5 per minute

 12:00 ─────────────── 12:01 ─────────────── 12:02
 | ● ● ● ● ●  ✗ ✗     | ● ● ●               |
 count = 5, then reject| counter resets to 0 |
```

- Dead simple and memory-cheap: one counter per key per window.
- **Edge problem:** bursts at the boundary. A client can send 5 at 12:00:59 and 5 more at 12:01:00 → **10 requests in 2 seconds**, double the intended rate.

```
            12:00                 12:01                 12:02
              |           ● ● ● ● ●|● ● ● ● ●           |
                                ↑ 10 requests in ~2 seconds ❌
```

### 4. Sliding window log

Store the **timestamp of every request**. On each new request, drop timestamps older than the window, then count what's left.

```
limit = 5 per 60 s, new request at 12:01:30

log: [12:00:25, 12:00:40, 12:01:05, 12:01:10, 12:01:20]
      ─ older than 12:00:30 → remove 12:00:25
remaining: 4 → under 5 → allow ✅ and add 12:01:30
```

- **Perfectly accurate** — no boundary problem.
- **Memory-heavy**: stores one entry per request. 10,000 req/min limit = up to 10,000 timestamps per user.

### 5. Sliding window counter

A hybrid: keep counters for the **current and previous fixed windows**, and estimate the count in the sliding window with a weighted sum.

```
limit = 100 per minute. Now it's 12:01:15 (15 s = 25% into the current window)

previous window (12:00-12:01): 80 requests
current  window (12:01-12:02): 30 requests so far

The sliding 60 s window still overlaps 75% of the previous window:

estimate = 30 + 80 × 0.75 = 90   → under 100 → allow ✅
```

- Smooths out the fixed-window boundary spike.
- Cheap: just two counters per key.
- An approximation (assumes requests in the previous window were evenly spread), but accurate enough in practice. Cloudflare has described using this approach at scale.

### Comparison

| Algorithm | Allows bursts? | Memory | Accuracy | Pros | Cons |
|---|---|---|---|---|---|
| **Token bucket** | ✅ Yes, up to bucket size | Low (2 values) | Good | Simple, burst-friendly, very popular | Two parameters to tune |
| **Leaky bucket** | ❌ No — smooth output | Low (queue size) | Good | Steady, predictable load downstream | Bursts get delayed/dropped; old requests can starve new ones |
| **Fixed window** | ⚠️ Up to 2x at window edges | Very low (1 counter) | Weak at boundaries | Easiest to build | Boundary burst problem |
| **Sliding window log** | ❌ No | High (every timestamp) | Exact | Precise | Memory cost grows with traffic |
| **Sliding window counter** | Mostly smoothed | Low (2 counters) | Close approximation | Good balance of accuracy and cost | Approximate |

**Default interview pick:** token bucket (or sliding window counter). Mention the fixed-window edge problem to show you understand the trade-offs.

---

## Distributed rate limiting with Redis

With one server, a counter in memory works. With **10 API servers behind a load balancer**, each has its own counter — a user could get 10x the limit by spreading requests across servers.

```
❌ Local counters                       ✅ Shared counter in Redis

User ──► LB ─┬─► S1 (count: 100)        User ──► LB ─┬─► S1 ─┐
             ├─► S2 (count: 100)                     ├─► S2 ─┼──► [ Redis ]
             └─► S3 (count: 100)                     └─► S3 ─┘    count: 100 total
 effectively 300 allowed                 exactly 100 allowed
```

Redis fits well: in-memory (fast), supports atomic operations, and keys can expire automatically.

**Fixed window in Redis:**

```
key = "rate:user42:202609301201"     ← user + current minute
INCR key                             ← atomic +1, returns new count
EXPIRE key 60                        ← (set when count == 1) auto-cleanup
if count > 100 → reject with 429
```

**Watch out for race conditions.** A "read the count, check it, then write it" sequence across two commands can let two servers both see 99 and both allow. Fixes:
- Use atomic commands (`INCR`).
- Run the whole check-and-update as a **Lua script** in Redis (executes atomically). This is how token bucket and sliding window are usually implemented.
- Sorted sets (`ZADD` / `ZREMRANGEBYSCORE` / `ZCARD`) are a common way to build a sliding window log.

Other considerations:
- **Redis becomes a dependency** — decide whether to **fail open** (allow requests if Redis is down, keep the site up) or **fail closed** (reject, stay safe). Most public APIs fail open.
- **Latency** — one Redis round trip per request; keep Redis close to the gateway.
- **Scale** — shard Redis by key when traffic is very large.

---

## HTTP 429 and rate-limit headers

When a client is over the limit, respond with **`429 Too Many Requests`** and tell them what's going on:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1790000000
```

| Header | Meaning |
|---|---|
| `Retry-After` | Seconds (or a date) to wait before retrying — a standard HTTP header |
| `X-RateLimit-Limit` | Max requests allowed in the window |
| `X-RateLimit-Remaining` | How many are left in the current window |
| `X-RateLimit-Reset` | When the window resets (often a Unix timestamp; some APIs use seconds remaining) |

The `X-RateLimit-*` headers are a widely used convention (GitHub, Twitter/X, and many others), not a formal standard; an IETF draft proposes standard `RateLimit` / `RateLimit-Policy` headers. Well-behaved clients should read these and back off, ideally with **exponential backoff + jitter** instead of retrying immediately.

---

## Where this shows up in HLD

- **"Design a rate limiter"** is a classic interview question on its own — expect to pick an algorithm, place it (API gateway), and make it distributed (Redis + atomic ops/Lua).
- In **any public API design** (payments, URL shortener, social APIs), mention rate limiting at the gateway as part of protecting the system.
- **Security-sensitive endpoints** (login, OTP, password reset) need tight per-user and per-IP limits against brute force.
- **Multi-tier limits** are common: per-IP at the edge, per-user/API key at the gateway, per-endpoint for expensive operations.
- Be ready to discuss **fail-open vs fail-closed**, and how clients should react to 429s.

---

## Key takeaways

- A rate limiter caps requests per client per time window to prevent abuse, ensure fairness, and protect systems and budgets.
- Enforce it server side — usually at the API gateway — keyed by user, API key, or IP.
- **Token bucket** allows controlled bursts; **leaky bucket** smooths output; **fixed window** is simple but leaks bursts at edges; **sliding window log** is exact but memory-heavy; **sliding window counter** is the cheap, accurate-enough middle ground.
- In a multi-server setup, keep counters in a shared store like **Redis**, updated atomically (INCR or Lua scripts).
- Reject with **HTTP 429** plus `Retry-After` and rate-limit headers so clients know when to try again.
