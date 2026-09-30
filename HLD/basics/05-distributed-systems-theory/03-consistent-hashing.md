# Consistent Hashing

## TL;DR

**Consistent hashing is a way to spread keys across servers so that adding or removing a server only moves a small slice of the keys.**

The naive approach, `hash(key) % N`, works until `N` changes. Add one server and **almost every key** suddenly maps to a different server — your cache goes cold, or your database has to reshuffle nearly everything.

Consistent hashing puts servers and keys on a circle (a "ring"). Each key goes to the next server clockwise. Add or remove a server, and only the keys next to it move — about **1/N of the keys** instead of nearly all of them.

---

## The problem with `hash(key) % N`

Say you have 4 cache servers and you pick one per key like this:

```
server = hash(key) % 4
```

Works great. Now traffic grows and you add a 5th server:

```
server = hash(key) % 5
```

| Key hash | `% 4` (before) | `% 5` (after) | Moved? |
|---|---|---|---|
| 10 | 2 | 0 | ❌ moved |
| 11 | 3 | 1 | ❌ moved |
| 12 | 0 | 2 | ❌ moved |
| 13 | 1 | 3 | ❌ moved |
| 14 | 2 | 4 | ❌ moved |
| 15 | 3 | 0 | ❌ moved |
| 16 | 0 | 1 | ❌ moved |
| 20 | 0 | 0 | ✅ stayed |

Going from N to N+1 servers moves roughly **N/(N+1)** of all keys — about **80%** here, and ~99% with 100 servers.

For a cache, that means nearly every lookup is now a miss, and all that traffic slams your database at once. For a sharded database, it means copying almost all your data between machines. The same thing happens when a server *dies*.

---

## The analogy that makes it click

**Houses on a circular road, and mail carriers stationed along it.**

Picture a round road. Houses (keys) are placed at spots along it. Mail carriers (servers) are stationed at a few spots too. The rule: **each house is served by the first carrier you meet walking clockwise.**

- A new carrier joins and stands somewhere on the road. They take over only the houses between them and the previous carrier. Everyone else keeps their carrier.
- A carrier quits. Only *their* houses move — to the next carrier clockwise. Nobody else notices.

Compare that to "house number % number of carriers" — hire one new carrier and almost every house gets reassigned.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Hash ring** | The output range of the hash function (e.g. 0 to 2³²−1) bent into a circle. |
| **Node** | A server placed on the ring by hashing its name/IP. |
| **Key** | A piece of data placed on the ring by hashing its key. |
| **Virtual node (vnode)** | One physical server placed on the ring at many positions. |
| **Rebalancing** | Moving keys when nodes join or leave. |

---

## How the ring works

### Step 1: Make a ring

Take a hash function with a fixed output range, say 0 to 2³²−1. Join the ends so the largest value wraps back to 0.

### Step 2: Place the servers

Hash each server's identifier (e.g. `hash("server-A")`) to get its spot on the ring.

### Step 3: Place the keys

Hash each key (e.g. `hash("user:42")`). Walk **clockwise** from that spot. The first server you hit owns the key.

```
                    Node A (top)
                  .─────●─────.
             .─'               '─.  ● k1
          .'                       '.
         /  ● k4                     \        clockwise = A → B → C → D → A
        |                             |
 Node D ●                             ● Node B   k1 → next clockwise → Node B
        |                             |          k2 → next clockwise → Node C
         \                     ● k2  /           k3 → next clockwise → Node D
          '.                       .'            k4 → next clockwise → Node A
       k3 ● '─.               .─'
                  '─────●─────'
                    Node C (bottom)
```

A simpler linear view (the right end wraps back to 0):

```
0 ─────────────────────────────────────────────────────── 2³²  (wraps to 0)
    k3   A      k1   k5    B      k2    C      k4    D     k6

  k3        → A     (first node to its right)
  k1, k5    → B
  k2        → C
  k4        → D
  k6        → nothing to its right, wraps around → A
```

Finding the owner is fast: keep node positions in a **sorted array** and binary search for the first position ≥ `hash(key)` (wrap to index 0 if none). That's O(log N).

---

## Adding a node: only neighbors move

Add **Node E** between A and B:

```
BEFORE:   ... A ──── k1 ──── k5 ──── B ...     B owns k1, k5

AFTER:    ... A ──── k1 ── E ── k5 ── B ...    E owns k1
                                                B still owns k5
```

- Only keys between **A and E** move (from B to E).
- Every key owned by C, D, and A stays put.
- On average, a new node takes about **1/N** of the keys.

## Removing a node: only its keys move

Node B dies:

```
BEFORE:   A ── [k1, k5] ── B ── [k2] ── C
AFTER:    A ── [k1, k5] ────────[k2] ── C     C now owns k1, k5, k2
```

Only B's keys move, all to the next node clockwise (C). Nobody else is affected.

| | `hash % N` | Consistent hashing |
|---|---|---|
| Keys moved when adding 1 node (N → N+1) | ~N/(N+1) (almost all) | ~1/(N+1) |
| Keys moved when a node dies | Almost all | Only that node's keys |
| Lookup cost | O(1) | O(log N) |
| Needs extra structure | No | Sorted list of node positions |

---

## The catch: uneven load

With only a few nodes, random positions on the ring are rarely evenly spaced:

```
     A  B                       C                                  D
 ────●──●───────────────────────●──────────────────────────────────●────
       ▲         ▲                           ▲                          ▲
    B owns   C owns a                    D owns a                   A owns
    a tiny   big arc                     HUGE arc                   (wraps)
    arc
```

Some servers get way more keys than others. Worse: when a node dies, **all** its load dumps onto **one** neighbor, which might then overload and die too (a cascade).

---

## Virtual nodes: the fix

Instead of placing each server on the ring once, place it **many times** — e.g. 100–256 positions per server, by hashing `"A#1"`, `"A#2"`, ... `"A#150"`.

```
Physical servers: A, B, C
Ring with 3 vnodes each (real systems use 100+):

 ─ A1 ── C1 ── B1 ── A2 ── B2 ── C2 ── A3 ── C3 ── B3 ─  (wraps)
```

Why this helps:

| Benefit | How |
|---|---|
| **Even load** | Many small arcs average out; each server ends up with close to 1/N of keys. |
| **Graceful failure** | When A dies, its many small arcs are spread across **many** different servers, not dumped on one. |
| **Different-sized machines** | Give a server with 2x the capacity 2x the vnodes. |
| **Smooth scaling** | A new server takes small slices from *every* existing server. |

The cost is a bigger lookup table (N × vnodes entries) — trivial in practice.

---

## Replication on the ring

Distributed databases usually store each key on **multiple** nodes. The common rule: store the key on the owner **plus the next R−1 distinct physical nodes clockwise**.

```
Replication factor 3:

   key k ──► Node B (primary) ──► Node C ──► Node D
             (skip vnodes of servers already chosen)
```

When a node dies, the replicas on its neighbors already have the data, so nothing is lost.

---

## Where it's used

| System | How it uses consistent hashing |
|---|---|
| **Amazon DynamoDB** (and the original Dynamo paper) | Partitions data across storage nodes; replicas are the next nodes on the ring. |
| **Apache Cassandra** | Token ring with vnodes (`num_tokens`); each node owns token ranges. |
| **Riak, ScyllaDB** | Same Dynamo-style ring. |
| **Distributed caches** (Memcached clients like ketama, Redis client-side sharding) | Pick which cache server holds a key so adding a server doesn't wipe the cache. |
| **CDNs** (Akamai's original paper, others) | Map URLs to edge cache servers within a PoP so each object lives on a predictable machine. |
| **Load balancers** (Envoy ring hash / Maglev, Nginx `hash ... consistent`) | Sticky routing: same user/session goes to the same backend, and minimal reshuffling when backends change. |
| **Discord, message brokers, chat systems** | Route a channel/user to a specific server. |

Note: **Redis Cluster** uses a related idea — 16,384 fixed **hash slots** assigned to nodes. Moving a slot moves only that slot's keys. Same goal, different mechanism.

---

## Where this shows up in HLD

- **"Design a distributed cache / key-value store."** Consistent hashing is the expected answer for "how do you decide which node stores a key?"
- **Sharding a database.** Explain how you add capacity without a massive reshuffle.
- **Scaling and failure questions.** "What happens when a node dies?" → only its keys move, and replicas on neighbors take over.
- **Hot spots.** Mention virtual nodes for balance — and that one *very* hot key still lands on one node (fix separately with caching or key splitting).
- **Sticky load balancing.** Routing the same user to the same server (for local caches or WebSocket connections) without breaking everything when a server is added.

---

## Key takeaways

- `hash(key) % N` remaps **almost all keys** whenever N changes — terrible for caches and shards.
- Consistent hashing places nodes and keys on a **ring**; each key belongs to the **next node clockwise**.
- Adding or removing a node moves only about **1/N of the keys** — just the ones next to it.
- **Virtual nodes** fix uneven load, spread a dead node's keys across many servers, and support different-sized machines.
- Replicas are usually the **next few distinct nodes clockwise**.
- Used in Cassandra, DynamoDB, distributed caches, CDNs, and load balancers.
