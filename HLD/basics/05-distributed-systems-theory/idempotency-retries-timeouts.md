# Idempotency, Retries & Timeouts

## TL;DR

**The network is unreliable. Requests get lost, replies get lost, servers get slow.**

Three tools work together to survive that:

- **Timeouts** — don't wait forever. Give up after a sensible time.
- **Retries** — try again, but politely: **exponential backoff + jitter**, and a limit.
- **Idempotency** — make it **safe** to retry, so doing something twice has the same effect as doing it once.

Retrying without idempotency is how customers get charged twice. Idempotency is what makes retries safe.

---

## The problem: you can't tell what went wrong

You send a request and get no answer. Which of these happened?

```
1. Request lost on the way      Client ──✗          Server      (nothing happened)
2. Server crashed mid-work      Client ─────────► Server  X      (maybe half done)
3. Server is just slow          Client ─────────► Server ...zzz  (will finish later)
4. Reply lost on the way back   Client      ✗──── Server ✅      (it DID happen!)
```

From the client's side, **all four look identical**: silence.

Case 4 is the dangerous one. The payment went through, but you don't know that. If you retry blindly, you pay twice. If you don't retry, in case 1 the payment never happens.

You can't fix the network. You can only design for it.

---

## The analogy that makes it click

**Ordering food by phone when the line keeps dropping.**

You call a restaurant, order a pizza, and the call cuts out before they confirm. Did they get the order?

- **Timeout:** You don't wait on hold for an hour. After 30 seconds of silence, you hang up.
- **Retry with backoff:** You call back — but if it's busy, you wait a bit longer each time instead of redialing every second.
- **Jitter:** If a thousand people lost the call at the same moment, you don't all redial at exactly the same second.
- **Idempotency key:** You say "this is order #A7 for Harsh." If they already have #A7, they say "got it already" instead of making a second pizza.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Timeout** | Max time to wait for a response before giving up. |
| **Retry** | Sending the same request again after a failure. |
| **Exponential backoff** | Waiting longer after each failed attempt (1s, 2s, 4s, 8s...). |
| **Jitter** | Random variation added to wait times so clients don't retry in sync. |
| **Idempotent** | Doing it once or many times gives the same final result. |
| **Idempotency key** | A unique ID the client attaches so the server can detect duplicates. |
| **Circuit breaker** | Stops calling a failing service for a while, to let it recover. |

---

## Timeouts

**Every network call needs a timeout.** Many libraries default to *no* timeout or a very long one, which means one slow dependency can freeze threads until your whole service is stuck.

### Types of timeouts

| Timeout | What it limits |
|---|---|
| **Connect timeout** | Time to open the TCP/TLS connection (should be short: ~1s). |
| **Read / request timeout** | Time waiting for the response after connecting. |
| **Overall deadline** | Total budget for the whole operation, including retries. |

### Choosing a value

- **Base it on real latency data**, not guesses. A common starting point: a bit above the dependency's **p99 latency** (e.g. p99 = 200 ms → timeout ≈ 300–500 ms).
- **Too short** → you give up on requests that would have succeeded, then retry them, adding load.
- **Too long** → threads and connections pile up waiting; your service goes down with the slow one.

### Deadline propagation

In a chain of services, timeouts must **shrink** as you go deeper:

```
User ─► API (deadline 2s) ─► Orders (remaining ~1.8s) ─► Payments (remaining ~1.5s)

❌ Bad: API times out at 2s, but Payments keeps working for 10s on a
       request nobody is waiting for anymore.
✅ Good: pass the remaining deadline downstream (gRPC does this built-in).
```

---

## Retries

Many failures are **transient**: a brief network blip, a node restarting, a momentary overload. Retrying often just works.

### What to retry — and what not to

| Retry ✅ | Don't retry ❌ |
|---|---|
| Timeouts, connection resets | `400 Bad Request` (it'll fail again) |
| `503 Service Unavailable` | `401/403` auth errors |
| `429 Too Many Requests` (respect `Retry-After`) | `404 Not Found` |
| `502/504` gateway errors | Validation errors, business-rule failures |

Rule: retry errors that might go away on their own. Never retry errors that are the client's fault.

### Exponential backoff

Don't retry immediately. Wait, and double the wait each time:

```
attempt 1 fails → wait 100 ms
attempt 2 fails → wait 200 ms
attempt 3 fails → wait 400 ms
attempt 4 fails → wait 800 ms
attempt 5 fails → give up, return error

wait = min(cap, base × 2^attempt)
```

This gives a struggling server breathing room, instead of hammering it while it's down.

### Jitter

Backoff alone has a flaw. If 10,000 clients fail at the same instant (e.g. the server restarted), they all wait exactly 100 ms, then all retry together, then all wait 200 ms and retry together again. Synchronized waves.

```
Without jitter:              With jitter:
 requests                     requests
   │█      █        █           │▅▃▄▅▃▄▃▄▅▃▄▃▅▄▃▄▃▅▄▃
   │█      █        █           │▅▃▄▅▃▄▃▄▅▃▄▃▅▄▃▄▃▅▄▃
   └────────────────── time     └──────────────────── time
   spikes crush the server      load spread out evenly
```

**Full jitter** (a popular choice): `wait = random(0, min(cap, base × 2^attempt))`.

### Retry storms

Retries **multiply load exactly when a system is weakest**.

```
         3 retries          3 retries          3 retries
Client ─────────► API ──────────► Service ──────────► Database
                                                       (slow)

One user request can become 3 × 3 × 3 = 27 requests to the database!
```

A small slowdown turns into an outage because everyone keeps retrying. How to prevent it:

- **Retry at one layer only** (usually the edge or the caller closest to the failure), not at every hop.
- **Cap attempts** (e.g. 3) and use an **overall deadline**.
- **Retry budgets**: allow retries only up to ~10% of normal traffic.
- **Circuit breakers** (below) to stop calling a service that's clearly down.

---

## Idempotency

An operation is **idempotent** if doing it **once or N times leaves the system in the same state**.

| Idempotent ✅ | Not idempotent ❌ |
|---|---|
| "Set balance to ₹500" | "Add ₹500 to balance" |
| "Delete order #42" | "Create a new order" |
| "Mark email as read" | "Send an email" |
| "Set status = SHIPPED" | "Increment view count" |

Only idempotent operations are safe to retry blindly.

### Idempotent HTTP methods

| Method | Idempotent? | Safe (no side effects)? | Note |
|---|---|---|---|
| `GET` | ✅ | ✅ | Just reads |
| `HEAD`, `OPTIONS` | ✅ | ✅ | |
| `PUT` | ✅ | ❌ | "Replace resource with this" — same result every time |
| `DELETE` | ✅ | ❌ | Deleting twice = still deleted (2nd may return 404, state is the same) |
| `POST` | ❌ | ❌ | "Create something" — twice creates two |
| `PATCH` | ❌ (not guaranteed) | ❌ | Depends: "set x=5" is, "append item" isn't |

Note: idempotency is about the **server state**, not the response. The second `DELETE` may return `404` instead of `200`, and that's fine.

### Idempotency keys: making POST safe

For operations like payments, you *need* `POST`, and you *need* retries. The fix: the client generates a unique ID per logical operation and sends it with every attempt.

```http
POST /payments
Idempotency-Key: 7f3a9c2e-1b4d-4e8a-9f6c-2d5e8a1b3c7f
{ "amount": 500, "to": "merchant_123" }
```

Server logic:

```
receive request with key K
   │
   ├─ K seen before, finished?   → return the SAVED response. Don't charge again.
   ├─ K seen before, in progress? → return 409 / "still processing"
   └─ K is new                    → store K as "in progress"
                                    process the payment
                                    save K + response (in the same transaction if possible)
                                    return response
```

```
Client ── POST (key=K) ──► Server: charges ₹500, saves K ✅
Client   ✗── reply lost ──
Client ── POST (key=K) ──► Server: "K already done" → returns same result
                           Customer charged once. ✅
```

Things to get right:

- The **client** generates the key (a UUID), once per user action — **not** per retry.
- Store keys with a **unique constraint** so two concurrent retries can't both win.
- Keep keys for a limited time (e.g. 24 hours, as Stripe does).
- Other ways to get the same effect: natural unique IDs (`order_id`), conditional writes (`UPDATE ... WHERE version = 3`), or deduplication tables for message consumers.

---

## Delivery guarantees

When sending messages (queues, events, webhooks), there are three possible promises:

| Guarantee | Meaning | How | Risk |
|---|---|---|---|
| **At-most-once** | Delivered 0 or 1 times | Send once, never retry | Messages can be **lost** |
| **At-least-once** | Delivered 1 or more times | Retry until acknowledged | **Duplicates** |
| **Exactly-once** | Processed effectively once | At-least-once **+ idempotent processing** (dedup) | Complexity |

```
At-most-once:    send ──✗   (lost, oh well)        → metrics, logs where loss is ok
At-least-once:   send ──✗ retry ──✓ retry ──✓       → most queues (SQS, Kafka default,
                              (processed twice!)       RabbitMQ with acks)
Exactly-once:    at-least-once + "have I seen msg_id 881? yes → skip"
```

**The honest truth:** true exactly-once *delivery* over an unreliable network is impossible. What systems (like Kafka's transactions) offer is **exactly-once *processing*** — duplicates may arrive, but their effect is applied once. In practice: **at-least-once delivery + idempotent consumers**. That's the pattern to state in interviews.

---

## Circuit breakers (briefly)

If a dependency is clearly down, retrying every request just wastes time and adds load. A circuit breaker **fails fast** instead.

```
        failures exceed threshold
 ┌────────┐ ───────────────────► ┌────────┐
 │ CLOSED │                      │  OPEN  │  → fail immediately, don't call
 │(normal)│ ◄──── success ─────┐ └────────┘
 └────────┘                    │     │ after a cool-down (e.g. 30s)
                               │     ▼
                           ┌───────────┐
                           │ HALF-OPEN │ → let a few test requests through
                           └───────────┘   fail → back to OPEN
```

While open, return a **fallback**: cached data, a default value, or a clear error. Libraries: Resilience4j (Java), Polly (.NET), Envoy/Istio (service mesh).

---

## Where this shows up in HLD

- **Payment systems.** "How do you avoid double charging?" → idempotency keys, stored with a unique constraint. This is asked constantly.
- **Message queues and event-driven designs.** State "at-least-once delivery with idempotent consumers" whenever you draw a queue.
- **Microservice calls.** Every arrow in your diagram needs a timeout. Mention retries with backoff + jitter and a circuit breaker for critical dependencies.
- **Handling failures / reliability questions.** Explain how retry storms turn a blip into an outage, and how you'd prevent it (retry at one layer, budgets, breakers).
- **API design.** Use `PUT` for idempotent updates; accept an `Idempotency-Key` header on `POST` endpoints that create money-moving or irreversible actions.

---

## Key takeaways

- When a request gets no answer, you **can't know** whether it happened. Design for that.
- **Always set timeouts**, based on real latency (around p99), and propagate deadlines downstream.
- Retry only transient errors, with **exponential backoff + jitter** and a hard cap — otherwise you cause **retry storms**.
- **Idempotency makes retries safe.** Use idempotent methods (`PUT`, `DELETE`) or **idempotency keys** for `POST` (e.g. payments).
- "Exactly-once" in practice = **at-least-once delivery + idempotent processing**.
- **Circuit breakers** fail fast when a dependency is down, giving it time to recover.
