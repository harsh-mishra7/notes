# Load Balancer

## Brief

**A load balancer (LB) sits in front of a group of servers and spreads incoming requests across them.**

One server can only handle so much. When traffic grows, you add more servers (horizontal scaling) — but clients need *one* address to talk to. The load balancer is that one address. It picks a healthy server for every request and forwards it.

It gives you two things at once:
- **Scalability** — add servers behind it, capacity goes up.
- **Availability** — if a server dies, the LB stops sending traffic to it.

---

## The problem it solves

Without a load balancer:

```
                 ┌──────────────┐
 1M users ─────► │  one server  │   ← CPU at 100%, crashes, everyone is down ❌
                 └──────────────┘
```

With a load balancer:

```
                                 ┌──────────┐
                           ┌───► │ server 1 │
                           │     └──────────┘
                ┌──────┐   │     ┌──────────┐
 1M users ────► │  LB  │ ──┼───► │ server 2 │
                └──────┘   │     └──────────┘
                           │     ┌──────────┐
                           └───► │ server 3 │   ← dies? LB skips it ✅
                                 └──────────┘
```

Users only know the LB's address (e.g. `api.myapp.com`). The servers behind it can be added, removed, or replaced without users noticing.

---

## The analogy that makes it click

**A load balancer is the host at a busy restaurant.**

Guests don't walk in and pick a random waiter. The host at the door looks at which waiters are free, which tables are open, and seats people evenly. If a waiter goes home sick, the host simply stops sending guests to their section.

The guests only ever deal with the host at the door. The kitchen can hire more waiters on a busy night, and no guest needs to know.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Backend / upstream / target** | A server sitting behind the LB that does the real work |
| **Server pool / target group** | The set of backends the LB can choose from |
| **Health check** | The LB regularly pinging backends to see if they're alive |
| **Algorithm** | The rule the LB uses to pick a backend |
| **Sticky session** | Sending the same user to the same backend every time |
| **VIP (Virtual IP)** | The single public IP clients connect to |

---

## L4 vs L7 load balancing

The "L" refers to layers of the OSI network model. It's about **how much of the request the LB looks at** before deciding where to send it.

```
L4 (Transport layer)                   L7 (Application layer)

Sees:  IP address + port               Sees:  full HTTP request
       TCP / UDP packets                      URL path, headers, cookies, method

"Something is coming in on             "This is  GET /api/users  with
 port 443 from 1.2.3.4"                  cookie session=abc"

Decision: forward the connection       Decision: /api → API servers
          to some server                          /images → image servers
```

| | **L4** | **L7** |
|---|---|---|
| Looks at | IP, port, protocol | URL, headers, cookies, body |
| Speed | Very fast, low overhead | Slightly slower (must parse HTTP) |
| Smart routing | ❌ No — can't see the URL | ✅ Yes — route by path, host, header |
| TLS | Usually passes it through untouched | Usually terminates it (decrypts) |
| Protocols | Any TCP/UDP (DBs, games, video) | Mostly HTTP/HTTPS, gRPC, WebSocket |
| Example | AWS NLB, HAProxy (TCP mode) | AWS ALB, Nginx, HAProxy (HTTP mode) |

**Rule of thumb:** use L7 for web apps and APIs (you want path-based routing). Use L4 when you need raw speed or the traffic isn't HTTP.

---

## Load balancing algorithms

How does the LB pick which server gets the next request?

### 1. Round Robin

Take turns, in order. Simple and the default almost everywhere.

```
req1 → S1   req2 → S2   req3 → S3   req4 → S1   req5 → S2 ...
```

Works great when all servers are identical and requests cost about the same.

### 2. Weighted Round Robin

Same idea, but bigger servers get more turns.

```
S1 (weight 3, big machine)    S2 (weight 1, small machine)

S1, S1, S1, S2, S1, S1, S1, S2, ...
```

Useful when your servers have different sizes, or when you slowly shift traffic to a new version (canary: 95% old, 5% new).

### 3. Least Connections

Send the request to whichever server has the **fewest active connections right now**.

```
S1: 12 open connections
S2:  3 open connections   ← next request goes here
S3:  8 open connections
```

Better than round robin when requests vary a lot in duration (some take 10 ms, some take 10 s), since round robin could pile long requests onto one server. There's also a *least response time* variant that factors in latency.

### 4. IP Hash

Hash the client's IP address, and use it to pick a server.

```
hash("1.2.3.4") % 3 = 1   → always S2
hash("5.6.7.8") % 3 = 0   → always S1
```

The same client always lands on the same server — a cheap form of stickiness. **Problem:** if you add or remove a server, `% 3` becomes `% 4`, and almost every client gets reshuffled to a different server.

### 5. Consistent Hashing

Fixes the reshuffling problem. Servers and keys are placed on a **ring**. Each key goes to the next server clockwise.

```
              S1
          ●───────●
        /           \
   key A             S2      key A → goes clockwise → S2
      |               |
       \             /       Add S4 between S2 and S3?
        ●───────────●        Only keys in that one slice move.
       S3        key B       Everything else stays put. ✅
```

When a server is added or removed, only about `1/N` of keys move instead of nearly all of them. This matters when a server holds state for a key — e.g. **caches** (so you don't lose your whole cache on a resize) or sharded data.

### Algorithm summary

| Algorithm | How it picks | Best for | Weakness |
|---|---|---|---|
| Round Robin | Takes turns | Identical servers, uniform requests | Ignores current load |
| Weighted RR | Turns, proportional to weight | Mixed server sizes, canary releases | Weights are static |
| Least Connections | Fewest active connections | Requests with very different durations | Needs to track state |
| IP Hash | hash(client IP) % N | Simple stickiness | Resizing reshuffles almost everyone |
| Consistent Hashing | Position on a hash ring | Caches, sharded/stateful backends | More complex; needs virtual nodes to spread evenly |

---

## Health checks

The LB needs to know which servers are alive. It checks them regularly.

```
Every 10 seconds:

LB ──── GET /health ────► S1   200 OK  ✅  keep in pool
LB ──── GET /health ────► S2   200 OK  ✅  keep in pool
LB ──── GET /health ────► S3   timeout ❌  (3 failures in a row → remove from pool)

... later ...
LB ──── GET /health ────► S3   200 OK  ✅  (2 successes in a row → add back)
```

| Type | How it works |
|---|---|
| **Active** | LB sends a probe (TCP connect, or HTTP `GET /health`) on a timer |
| **Passive** | LB watches real traffic — if a server keeps returning errors/timeouts, mark it bad |

**Tip:** a good `/health` endpoint checks that the app can actually do its job (e.g. can reach its database), not just that the process is running. But don't make it so strict that one slow dependency takes every server out at once.

---

## Sticky sessions (session affinity)

Some apps store user session data **in the memory of one server**. If the user's next request goes to a different server, that server doesn't know them — they appear logged out.

Sticky sessions fix this by pinning a user to one server, usually with a cookie the LB sets:

```
First request:   User ──► LB ──► S2        LB sets cookie:  lb=S2
Later requests:  User (cookie lb=S2) ──► LB ──► S2   (always)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Works with apps that keep state in memory | Load becomes uneven (one server gets the "heavy" users) |
| No code changes needed | If S2 dies, those users lose their session anyway |
| | Makes scaling down and deploys harder |

**The better fix:** make servers **stateless**. Store sessions in a shared store (Redis, a database) or in a signed token (JWT). Then any server can handle any request, and you don't need stickiness at all.

---

## "But isn't the load balancer a single point of failure?"

Yes — if you have exactly one. If it dies, everything behind it is unreachable. So in practice you never run just one.

```
                       ┌──────────────┐
                ┌────► │  LB (active) │ ────► servers
                │      └──────────────┘
 clients ──► VIP        heartbeat ↕
                │      ┌──────────────┐
                └ ─ ─► │ LB (standby) │   takes over the VIP if active dies
                       └──────────────┘
```

Common approaches:

| Approach | How it works |
|---|---|
| **Active-passive** | A standby LB watches the active one (heartbeat). If active dies, standby grabs the virtual IP (e.g. using keepalived / VRRP). |
| **Active-active** | Several LBs all serve traffic. DNS returns multiple IPs, or anycast routes users to the nearest one. |
| **Managed cloud LB** | AWS ALB/NLB, GCP LB, etc. are already redundant across machines and availability zones. You don't manage failover yourself. |

Big systems often stack them: **DNS → L4 LB → L7 LB → app servers.**

---

## Common load balancers

| Tool | Type | Notes |
|---|---|---|
| **Nginx** | Software, mostly L7 (also L4 via `stream`) | Also a web server and reverse proxy. Very common. |
| **HAProxy** | Software, L4 and L7 | Known for high performance and detailed stats |
| **Envoy** | Software, L4 and L7 | Popular in microservices and service meshes |
| **AWS ALB** | Managed, L7 | HTTP/HTTPS routing by path/host, WebSockets, gRPC |
| **AWS NLB** | Managed, L4 | TCP/UDP, extremely high throughput, static IPs |
| **GCP / Azure LB** | Managed, L4 and L7 | Cloud equivalents |
| **F5, Citrix** | Hardware appliances | Older enterprise data centers |
