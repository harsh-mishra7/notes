# Sharding and Partitioning

## TL;DR

**Partitioning = splitting one big dataset into smaller pieces.**
**Sharding = horizontal partitioning where each piece lives on a different machine.**

When one database server can't hold all the data or handle all the writes, you split the rows across several servers. Each server (a **shard**) owns a subset of rows, for example users A-M on shard 1 and N-Z on shard 2.

It's the main way to scale **writes** and **storage**. It's also one of the most painful things to change later, so choosing the **shard key** is the big decision.

---

## The analogy that makes it click

**A library that has run out of shelf space.**

One building can't hold every book anymore, so you open more branches.

- **Vertical split** → Branch 1 holds all fiction, Branch 2 holds all science. (Split by *kind* of data.)
- **Horizontal split (sharding)** → every branch holds every kind of book, but Branch 1 gets authors A-M and Branch 2 gets N-Z. (Split by *rows*.)

The rule you use to decide "which branch has this book?" is your **shard key** + **sharding strategy**.

And if you ever want to reorganize from 2 branches to 5... you're moving a *lot* of books. (**Resharding**)

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Partition** | One piece of a split dataset. |
| **Shard** | A partition living on its own machine (or node). |
| **Shard key / partition key** | The column used to decide which shard a row goes to. |
| **Hot spot / hot shard** | A shard getting far more traffic than the others. |
| **Resharding / rebalancing** | Moving data around when you add or remove shards. |
| **Scatter-gather** | Sending a query to all shards and merging results. |

---

## Vertical vs horizontal partitioning

```
Original table: users (id, name, email, bio, avatar_blob, settings_json)

VERTICAL (split columns)                HORIZONTAL (split rows) = SHARDING
┌────────────────────┐ ┌──────────────┐  ┌──────────────────────────┐
│ id, name, email    │ │ id, bio,     │  │ Shard 1: users 1 - 1M    │
│ (hot, small)       │ │ avatar_blob  │  ├──────────────────────────┤
│                    │ │ (cold, big)  │  │ Shard 2: users 1M - 2M   │
└────────────────────┘ └──────────────┘  ├──────────────────────────┤
                                         │ Shard 3: users 2M - 3M   │
                                         └──────────────────────────┘
```

| | Vertical partitioning | Horizontal partitioning (sharding) |
|---|---|---|
| **Splits** | Columns (or whole tables/features) | Rows |
| **Each piece has** | Different columns, all rows | Same columns, different rows |
| **Example** | Users table on DB 1, Orders on DB 2 | Users 1-1M on DB 1, 1M-2M on DB 2 |
| **Scales** | Only until one table outgrows a machine | Nearly without limit |
| **Complexity** | Lower | Higher |

Vertical splitting by feature (users DB, orders DB, payments DB) is often the natural first step, and is what happens with microservices. Sharding comes when a single table is too big.

> Don't shard too early. First try: better indexes, caching, read replicas, a bigger machine. Sharding adds permanent complexity.

---

## Sharding strategies

### 1. Range-based

Each shard owns a **continuous range** of the key.

```
user_id 1        - 1,000,000   → Shard 1
user_id 1,000,001 - 2,000,000  → Shard 2
created_at Jan-Mar             → Shard 1
created_at Apr-Jun             → Shard 2
```

- ✅ Simple. Range queries (`WHERE id BETWEEN ...`) hit just one or two shards.
- ❌ Easily creates hot spots: with time-based ranges, **all new writes land on the newest shard**.

### 2. Hash-based

Hash the key, then map the hash to a shard.

```
shard = hash(user_id) % number_of_shards

hash(42)  % 4 = 2  → Shard 2
hash(43)  % 4 = 0  → Shard 0
```

- ✅ Spreads data and load evenly.
- ❌ Range queries must hit every shard (neighbors are scattered).
- ❌ Plain `% N` is terrible when N changes: going from 4 to 5 shards moves ~80% of keys. **Consistent hashing** fixes this (see below).

### 3. Directory / lookup-based

A separate **lookup service** stores "key → shard" explicitly.

```
┌──────────────────────┐
│ Lookup table         │
│ tenant_acme  → S1    │
│ tenant_globex→ S3    │
│ tenant_initech→ S2   │
└──────────────────────┘
```

- ✅ Total flexibility: move one big customer to its own shard whenever you want.
- ❌ The lookup service is an extra hop and a critical dependency (must be cached and highly available).

### 4. Geo-based

Shard by the user's **region**.

```
India users   → Mumbai shard
EU users      → Frankfurt shard
US users      → Virginia shard
```

- ✅ Low latency (data near users) and helps with data-residency laws (e.g. GDPR).
- ❌ Uneven load (regions differ in size); users who travel or move need handling.

### Comparison

| Strategy | Even distribution | Range queries | Easy to add shards | Typical use |
|---|---|---|---|---|
| **Range** | ❌ Risk of hot spots | ✅ | ~ Split ranges | Time-series, ordered IDs |
| **Hash** | ✅ | ❌ | ❌ with `% N`, ✅ with consistent hashing | User data, general purpose |
| **Directory** | ✅ You control it | Depends | ✅ | Multi-tenant SaaS |
| **Geo** | ❌ Often uneven | ✅ within region | ~ | Global apps, compliance |

---

## Choosing a shard key

This is the most important decision. A good shard key:

1. **Has high cardinality** — many distinct values (user_id ✅, country ❌, boolean ❌❌).
2. **Spreads load evenly** — no single value gets most of the traffic.
3. **Matches your most common query** — so most queries hit **one** shard.
4. **Rarely changes** — changing a row's shard key means moving the row.

Example — a chat app:

| Shard key | Good or bad? | Why |
|---|---|---|
| `message_id` | ❌ | Loading one conversation hits every shard |
| `created_at` | ❌ | All current writes go to one shard |
| `chat_id` | ✅ | A conversation's messages live together; reads hit one shard |
| `user_id` | ~ | Group chats span many users → cross-shard reads |

**Rule of thumb:** shard by the thing your main query is "about" — `WHERE user_id = ?` → shard by `user_id`.

---

## Hot spots and the celebrity problem

Even with hashing, **one key can be hot**. Hashing spreads *keys* evenly, not *traffic*.

```
Shard 1: [ random users ... ]            ░░░░ normal load
Shard 2: [ celebrity with 100M followers ] ████████████████ overloaded
Shard 3: [ random users ... ]            ░░░░ normal load
```

A celebrity posts, and millions of reads/likes hit the one shard holding that key.

Fixes:

- **Key salting / splitting** — store the hot key as `celebrity_id#0 ... celebrity_id#9` across 10 shards; write to a random one, read from all and combine.
- **Caching** — put hot data in a cache (Redis/CDN) in front of the shard.
- **Dedicated shard** — with directory-based sharding, move the celebrity to their own machine.
- **Different handling for big accounts** — e.g. in a news feed, don't fan out a celebrity's posts to every follower; merge them in at read time.

---

## Resharding: the pain

You started with 4 shards. You now need 8. Every shard key rule changes, and lots of data must physically move — **while the system keeps serving traffic**.

```
hash(key) % 4   →   hash(key) % 8
Roughly half the keys now belong to a different shard. Move them all, live.
```

What makes it hard:

- Moving terabytes takes hours or days.
- Writes keep arriving for data that's mid-move.
- Routing must switch over without downtime or lost writes.

How systems make it bearable:

- **Consistent hashing** — adding a node only moves ~1/N of keys (its neighbors' share), not most of them.
- **Many small virtual partitions** — create e.g. 1024 logical partitions up front, map them to 4 machines. To scale, just move whole partitions to new machines; the key → partition mapping never changes. (Used by Cassandra's vnodes, Elasticsearch, Couchbase, DynamoDB internally.)
- **Double writes + backfill** — write to old and new locations during migration, then cut over.

---

## Cross-shard queries and joins

Once data is split, anything that needs data from multiple shards gets expensive.

```
Query: "Top 10 highest-spending users overall"

App ──► Shard 1 ──► top 10 ┐
    ──► Shard 2 ──► top 10 ├──► merge ──► final top 10     (scatter-gather)
    ──► Shard 3 ──► top 10 ┘
```

| Problem | Why it hurts | Common fix |
|---|---|---|
| **Joins across shards** | Rows are on different machines | Denormalize; co-locate related data by the same shard key |
| **Scatter-gather queries** | Latency = slowest shard; load × N | Avoid for hot paths; precompute |
| **Global secondary indexes** | "Find user by email" when sharded by user_id | Separate index table sharded by email |
| **Transactions across shards** | Needs 2-phase commit, slow and fragile | Keep transactions within one shard; use sagas |
| **Unique constraints / auto-increment IDs** | No single place to check | Global ID generators (Snowflake IDs, UUIDs) |
| **Analytics** | Need all the data | Copy to a data warehouse |

**Design principle:** choose the shard key so that **data that's used together lives together** (e.g. a user's orders on the user's shard).

---

## Link to consistent hashing

Plain `hash(key) % N` breaks badly when N changes. **Consistent hashing** places both keys and servers on a ring; each key belongs to the next server clockwise. Adding or removing a server only affects the keys next to it.

```
          Server A
        ●
   k1 ·     · k2
 ●              ● Server B
Server D   ·k3
        ●
      Server C

Add Server E between A and B → only keys between A and E move to E.
```

It's covered in depth in [consistent-hashing.md](../05-distributed-systems-theory/03-consistent-hashing.md); the key point here is that it makes **adding and removing shards cheap**, which is why Cassandra, DynamoDB, and many caches use it.

---

## Where this shows up in HLD

- Almost every "design X at scale" question reaches: "one database can't handle this — how do you shard?" Name the **shard key** and **why**.
- Back-of-envelope math decides it: e.g. 10 TB of data at 1 TB per machine → ~10+ shards.
- Interviewers probe the weak spots: hot keys (celebrities), cross-shard queries, and resharding. Have an answer for each.
- Many managed databases shard for you (DynamoDB, Cassandra, MongoDB, Vitess for MySQL, Citus for Postgres) — but the **key choice is still yours**.

---

## Key takeaways

- Vertical partitioning splits columns/tables; horizontal partitioning (sharding) splits rows across machines.
- Strategies: range (good for ranges, risks hot spots), hash (even, no ranges), directory (flexible, extra hop), geo (latency, compliance).
- The shard key should be high-cardinality, evenly used, and match your main query.
- Hot keys need salting, caching, or special handling; hashing alone doesn't fix them.
- Resharding and cross-shard queries are the real costs — consistent hashing and co-locating related data reduce them.
