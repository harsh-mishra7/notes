# Stateless vs Stateful Servers

## TL;DR

**State** = anything the server remembers about you *between* requests (who you are, what's in your cart, what step of checkout you're on).

- A **stateful** server keeps that memory **inside itself**. You must keep talking to that same server.
- A **stateless** server keeps **nothing** between requests. Every request carries what's needed, or the server looks it up in a shared store.

Stateless servers are interchangeable, so you can add or remove them freely. That's why "make the app tier stateless" is one of the most repeated rules in system design.

---

## The analogy that makes it click

**Two kinds of help desks.**

- **Stateful: your personal banker.** Priya knows your history. Great, until Priya is on vacation, or there's a queue of 200 people all waiting for *their* banker. Nobody else can help you.
- **Stateless: a ticket counter with a shared computer.** Any clerk can help you because your details are in the **shared system** (or on the **ticket you carry**). One clerk goes home? Next clerk. Rush hour? Open more counters.

The information didn't disappear. It just moved **out of the person's head** into a place everyone can reach.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **State** | Data remembered across requests (session, cart, progress) |
| **Session** | The state tied to one logged-in user's visit |
| **Stateful server** | Keeps session data in its own memory |
| **Stateless server** | Keeps no per-client data between requests |
| **Horizontal scaling** | Adding more machines (scale *out*) |
| **Sticky session** | The load balancer always sends a user to the same server |
| **External state store** | A shared place for state (Redis, database) outside the app servers |

---

## What "state" actually means

Not all state is the problem. Separate it:

| Kind of state | Example | Where it should live |
|---|---|---|
| **Durable data** | Users, orders, posts | Database (always) |
| **Session state** | "User 42 is logged in", cart, wizard step | Shared store (Redis) or in a token the client carries |
| **In-memory state on the server** | A `sessions = {}` map in your Node process | ❌ This is what makes a server stateful |
| **Caches** | Recently fetched products | Fine locally *if* losing it is harmless; shared cache otherwise |

"Stateless server" doesn't mean "the system has no state". It means **the app servers don't hold state that's needed for the next request.** The state moves somewhere shared.

---

## The problem with stateful servers

Here's a stateful server storing sessions in memory:

```
                     ┌──────────────────────┐
User A logs in ────► │ Server 1             │
                     │ sessions: { A: ... } │
                     └──────────────────────┘
                     ┌──────────────────────┐
                     │ Server 2             │
                     │ sessions: { }        │
                     └──────────────────────┘
```

Now put a load balancer in front, which spreads requests around:

```
Request 1 from A ──► LB ──► Server 1   "Hi A, you're logged in" ✅
Request 2 from A ──► LB ──► Server 2   "Who are you? Please log in" ❌
```

User A gets randomly logged out. Adding servers **broke** the app instead of scaling it.

---

## Fix #1 (the band-aid): sticky sessions

Tell the load balancer to **always send the same user to the same server**, usually using a cookie or the client's IP.

```
User A ──► LB ──(always)──► Server 1
User B ──► LB ──(always)──► Server 2
User C ──► LB ──(always)──► Server 1
```

It works. But it has real problems:

| Problem | What happens |
|---|---|
| **Server dies** | Everyone "stuck" to it loses their session. Logged out, cart gone. ❌ |
| **Uneven load** | A few heavy users on Server 1 overload it while Server 2 is idle. The LB can't rebalance. ❌ |
| **Scaling in/out is messy** | Removing a server kicks its users; a new server gets no existing users, only new ones. ❌ |
| **Deploys hurt** | Restarting a server for a release drops its sessions. ❌ |
| **IP-based stickiness breaks** | Many users behind one office/mobile NAT IP all land on one server. ❌ |

Sticky sessions are OK as a quick fix, or when state is truly hard to move (e.g. a long-lived WebSocket connection). But they're not a scaling strategy.

---

## Fix #2 (the real fix): move state out of the server

### Option A: a shared session store (Redis)

```
                   ┌──────────┐
             ┌───► │ Server 1 │ ───┐
User A ──► LB      └──────────┘    │    ┌────────────────────┐
             │     ┌──────────┐    ├──► │ Redis              │
             ├───► │ Server 2 │ ───┤    │ session:abc → A    │
             │     └──────────┘    │    └────────────────────┘
             │     ┌──────────┐    │
             └───► │ Server 3 │ ───┘
                   └──────────┘
```

1. User logs in. Server creates a session ID `abc`, stores `abc → user A` in Redis, and sends `abc` back in a cookie.
2. Next request goes to **any** server. It reads the cookie, looks up `abc` in Redis, and knows it's A.

Why Redis? It's in-memory (sub-millisecond reads), supports expiry (TTL) so sessions auto-expire, and can be replicated for availability. A database also works, just slower.

### Option B: put the state in the request itself (tokens)

The server signs a token (like a **JWT**) containing "this is user 42, expires at 5pm" and hands it to the client. The client sends it on every request. Any server can verify the signature, **no lookup needed**.

```
Client ──► [ Authorization: Bearer eyJhbGc... ] ──► any server ── verifies signature ✅
```

Trade-off: you can't easily "un-issue" a token before it expires. (Details in `authentication-and-authorization.md`.)

### Option C: keep it on the client

Small, non-sensitive UI state (theme, draft text, current tab) can just live in the browser (localStorage, URL query params). The server never needs it.

---

## Stateful vs stateless, side by side

| | Stateful server | Stateless server |
|---|---|---|
| **Where session lives** | Server's memory | Redis / DB / token on client |
| **Any server can handle any request?** | ❌ | ✅ |
| **Horizontal scaling** | Hard | Easy: just add servers |
| **Load balancing** | Needs sticky sessions | Any algorithm (round-robin, least-connections) |
| **Server crash** | Users lose sessions | Nothing lost; LB routes around it |
| **Deploys / autoscaling** | Disruptive | Painless |
| **Per-request cost** | Lower (data already in memory) | Slightly higher (lookup or verify token) |
| **Complexity** | Simple at first | Needs an extra store or token logic |

---

## Why stateless servers scale horizontally

When servers are identical and hold nothing unique:

```
Traffic doubles?     → add servers.     No migration needed.
A server crashes?    → LB stops using it. Users don't notice.
Traffic drops?       → remove servers.  Nobody gets logged out.
New release?         → rolling restart. Zero downtime.
```

This is what "cattle, not pets" means. Pets (stateful servers) are unique and you nurse them when they're sick. Cattle (stateless servers) are identical and replaceable.

**But the state didn't vanish.** You moved the hard problem into Redis or the database. Those now need their own scaling and replication. That's fine: it's much easier to scale one specialized store than to make every app server special.

---

## When stateful is actually the right call

Some systems are stateful by nature:

- **Databases and caches** themselves (Postgres, Redis, Kafka). Their whole job is state.
- **WebSocket / real-time servers** (chat, multiplayer games). Each open connection lives on one specific server. You usually keep a mapping like "user 42 is connected to server 7" in a shared store so messages can be routed there.
- **Game servers** holding a live match in memory for speed.

For these, you design explicitly for it: replication, partitioning, and routing to the right node.

---

## Gotchas

- **Local file uploads are state too.** Saving uploads to a server's disk breaks when the next request hits another server. Use object storage (S3).
- **In-memory caches cause inconsistency.** Server 1 caches old data, Server 2 has new data. Users see different results. Use a shared cache or short TTLs.
- **Scheduled jobs on every server.** Running a cron inside each app server means the job runs N times. Use a single scheduler or a distributed lock.
- **The session store becomes critical.** If Redis goes down, everyone is logged out. Replicate it.
- **Don't put secrets in client-side state.** Tokens can be read by the client; don't store anything there you wouldn't want the user to see (unless encrypted).

---

## Where this shows up in HLD

- Almost every design has **"stateless app servers behind a load balancer"**. Say it explicitly, and say where the state goes (Redis for sessions, DB for data, S3 for files).
- When the interviewer asks **"how would you scale this?"**, stateless app tier is the first answer: add servers and autoscale.
- **Sticky sessions** come up as a trap: know why they hurt availability and load balancing.
- **Chat, notifications, live location, multiplayer** designs are stateful by nature. Expect questions like "user A is on server 3 and user B is on server 9, how does a message get from A to B?" (Answer: a shared connection registry plus pub/sub.)
- **Auth design** is tied to this: session store vs JWT is really a stateful-vs-stateless choice.

---

## Key takeaways

- **State** is what a server remembers between requests. If it's in the server's memory, the server is stateful.
- **Stateless servers are interchangeable**, which makes horizontal scaling, failover, and deploys easy.
- **Sticky sessions** patch the problem but hurt reliability and load balancing.
- The real fix: **move state out**: sessions to Redis, data to the DB, files to object storage, or into a signed token.
- The state doesn't disappear; it moves to **specialized stores** that you scale and replicate on purpose.
- Some things (DBs, WebSocket servers) are stateful by design. Plan routing and replication for them.
