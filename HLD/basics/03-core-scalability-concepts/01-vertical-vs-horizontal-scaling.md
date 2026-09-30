# Vertical vs Horizontal Scaling

## Brief

Your app is getting more traffic than one server can handle. You have two choices:

- **Vertical scaling (scale up)** — make the one machine **bigger**: more CPU, more RAM, faster disk.
- **Horizontal scaling (scale out)** — add **more machines** and split the work between them.

Scaling up is simple but hits a ceiling and leaves you with a single point of failure.
Scaling out has almost no ceiling, but your app has to be *designed* for it.

Most real systems do both: scale up until it gets expensive or awkward, then scale out.

---

## The analogy that makes it click

**You run a restaurant kitchen that can't keep up with orders.**

- **Vertical:** replace your cook with a faster, more experienced chef and buy a bigger stove. Easy — nothing about how the kitchen works changes. But there's only so fast one human can cook, and top chefs get very expensive. And if that chef calls in sick, the kitchen is closed.
- **Horizontal:** hire five ordinary cooks. Now you can handle far more orders, and if one is sick the others keep going. But you need someone to hand out orders (a **load balancer**), every cook needs the same recipes (**stateless servers**), and they can't all fight over one fridge (**data partitioning**).

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Scale up / vertical** | Upgrade a single machine's resources |
| **Scale out / horizontal** | Add more machines (nodes / instances) |
| **Scale in / scale down** | The reverse — remove machines, or shrink one |
| **Node / instance** | One server in a group of servers |
| **Stateless** | The server keeps no user-specific data between requests |
| **Load balancer** | Spreads incoming requests across nodes |

---

## What each looks like

```
VERTICAL (scale up)                    HORIZONTAL (scale out)

  ┌────────┐        ┌──────────────┐          ┌──────────────┐
  │ 4 CPU  │  ───►  │   64 CPU     │          │ Load Balancer│
  │ 16 GB  │        │   512 GB     │          └──────┬───────┘
  └────────┘        │   NVMe SSD   │        ┌───────┼───────┐
                    └──────────────┘        ▼       ▼       ▼
                                         ┌─────┐ ┌─────┐ ┌─────┐
   same box, bigger                      │ S1  │ │ S2  │ │ S3  │  ... S100
                                         └─────┘ └─────┘ └─────┘
                                           many ordinary boxes
```

---

## Vertical scaling: pros and limits

**Why people like it:**

- **Zero code changes.** Your app doesn't know it's on a bigger box.
- **No distributed-systems problems.** One machine means no network calls between nodes, no data sync, no consistency headaches.
- **Great for databases.** A single relational database on a big machine goes a very long way.

**Where it breaks:**

- **Hard ceiling.** The biggest cloud machines top out at a few hundred CPUs and a few TB of RAM. You cannot buy past that.
- **Cost grows faster than capacity.** A machine with 2x the power often costs *more* than 2x. High-end hardware is priced at a premium.
- **Single point of failure.** One box dies, the whole service is down.
- **Downtime to upgrade.** Resizing a VM usually means a restart.

```
Cost
 │                              ╱   vertical: gets steep
 │                           ╱
 │                       ╱
 │                 ╱
 │          ╱ ─ ─ ─ ─ ─ ─ ─ ─ ─   horizontal: roughly linear
 │    ╱ ─ ─
 │ ╱─
 └────────────────────────────── Capacity
                       ▲
                  hardware ceiling
```

---

## Horizontal scaling: pros and costs

**Why people like it:**

- **Near-unlimited capacity.** Need more? Add more nodes.
- **Fault tolerance.** One node dies, the others carry the load.
- **Linear-ish cost.** Commodity machines are cheap; you pay per node.
- **No-downtime upgrades.** Roll new code out one node at a time.

**What it costs you:**

- **Complexity.** You now run a distributed system — networks fail, nodes disagree, clocks drift.
- **Your app must be built for it.** See the next section.
- **Operational overhead.** More machines to deploy, monitor, patch, and pay for.

---

## What horizontal scaling *requires*

You can't just copy your server three times and hope. Three things have to be true.

### 1. Stateless application servers

If Server 1 stores Alice's session in its own memory, and her next request lands on Server 2, she's suddenly logged out.

```
❌ Stateful                             ✅ Stateless

Alice ──► S1 (has her session)          Alice ──► S1 ─┐
Alice ──► S2 (who are you?)             Alice ──► S2 ─┼──► Redis / DB
                                                      │    (sessions live here)
                                        Alice ──► S3 ─┘
```

**Fix:** move state out of the server — into a shared cache (Redis), a database, or a signed token the client carries (JWT). Now any server can handle any request, and servers become interchangeable.

> "Sticky sessions" (the load balancer always sends Alice to S1) is a workaround, not a fix. When S1 dies, Alice's session dies with it, and load gets uneven.

### 2. Load balancing

Something has to sit in front and decide which server gets each request.

| Strategy | How it picks |
|---|---|
| Round robin | Next server in the list, in turn |
| Least connections | The server with the fewest active requests |
| IP / consistent hash | Same client (or key) always goes to the same server |
| Weighted | Bigger servers get a bigger share |

The load balancer also runs **health checks** and stops sending traffic to dead nodes.

### 3. Data partitioning (for the data layer)

App servers are easy to scale once they're stateless. **The database is the hard part** — you can't just run five copies that all accept writes without them disagreeing.

Common techniques:

| Technique | What it does | Helps with |
|---|---|---|
| **Read replicas** | Copies of the DB that serve reads; one primary takes writes | Read-heavy traffic |
| **Caching** | Keep hot data in Redis/Memcached | Repeated reads |
| **Sharding / partitioning** | Split data across DBs by a key (e.g. `user_id % 4`) | Write-heavy traffic, huge datasets |

```
          user_id % 3
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 ┌───────┐ ┌───────┐ ┌───────┐
 │Shard 0│ │Shard 1│ │Shard 2│
 │ 0,3,6 │ │ 1,4,7 │ │ 2,5,8 │
 └───────┘ └───────┘ └───────┘
```

Sharding is powerful but painful: cross-shard queries and joins get hard, and rebalancing when you add a shard is real work. Most teams delay it as long as they can.

---

## Auto-scaling

Horizontal scaling unlocks a bonus: **the number of servers can change on its own** based on load.

```
Traffic   ▁▁▂▃▅▇█▇▅▃▂▁▁
Servers   2 2 2 4 6 8 8 8 6 4 2 2 2
                ▲             ▲
          scale out       scale in
         (CPU > 70%)     (CPU < 30%)
```

**How it works:** you set rules — "keep average CPU around 60%", "min 2, max 20 instances" — and the platform (AWS Auto Scaling Groups, Kubernetes HPA, GCP Managed Instance Groups) adds or removes instances.

| Type | Trigger | Example |
|---|---|---|
| **Reactive** | A metric crosses a threshold | CPU > 70% for 5 min → add 2 nodes |
| **Scheduled** | A known time | Scale up every weekday at 9 AM |
| **Predictive** | Forecast from past traffic | ML-based, e.g. AWS predictive scaling |

**Things to watch:**

- **New instances take time to boot** (seconds to minutes). A sudden spike can overwhelm you before help arrives. Keep some headroom.
- **Scale in slowly.** Removing nodes too eagerly causes "flapping" (up, down, up, down).
- **Only works if servers are stateless.** Killing a node must not lose anything.
- **The database usually doesn't auto-scale** the same way. Adding 50 app servers can just move the bottleneck to the DB.

---

## Side-by-side comparison

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| **How** | Bigger machine | More machines |
| **Ceiling** | Hard hardware limit ❌ | Practically none ✅ |
| **Code changes** | None ✅ | App must be stateless, data partitioned ❌ |
| **Complexity** | Low ✅ | High (distributed system) ❌ |
| **Fault tolerance** | Single point of failure ❌ | Survives node failures ✅ |
| **Cost curve** | Gets steep at the top ❌ | Roughly linear ✅ |
| **Upgrade downtime** | Usually needs a restart ❌ | Rolling, zero downtime ✅ |
| **Auto-scaling** | Awkward ❌ | Natural fit ✅ |
| **Data consistency** | Easy (one copy) ✅ | Hard (many copies) ❌ |
| **Best for** | Early stage, databases, simple apps | Web/app tiers, large scale, high availability |

---

## A realistic path

```
Stage 1:  1 server (app + DB)                 ← start here, it's fine
Stage 2:  Bigger server                        ← vertical, cheap win
Stage 3:  Separate app server and DB server
Stage 4:  Multiple stateless app servers + LB  ← horizontal for app tier
Stage 5:  DB read replicas + cache
Stage 6:  Shard the DB                          ← only when truly needed
```

Scale the **stateless** parts horizontally early (it's cheap). Scale the **stateful** parts (the database) vertically for as long as you reasonably can.
