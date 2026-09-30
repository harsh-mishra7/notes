# Database Replication

## Brief

**Replication = keeping copies of the same data on multiple machines.**

Why? So that if one machine dies you don't lose data or go down, and so that many machines can share the read traffic.

The most common setup is **leader-follower**: one machine accepts writes, and copies them to others that serve reads. The hard part is that copies take time to catch up — this delay is **replication lag**, and it causes subtle bugs.

---

## The analogy that makes it click

**A teacher and students copying notes.**

The teacher (the **leader**) writes on the board. Students (the **followers**) copy it into their notebooks.

- If you want the *official* answer, ask the teacher.
- If you just want to read the notes, ask any student — spreads the load.
- Some students write slower than others. Ask a slow one and you'll get notes that are a few lines behind. (**Replication lag**)
- If the teacher leaves, one student is promoted to teacher. (**Failover**)

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Leader / primary / master** | The node that accepts writes. |
| **Follower / replica / secondary / slave** | A node that copies data from the leader. |
| **Replication log** | The stream of changes the leader sends to followers. |
| **Replication lag** | How far behind a follower is from the leader. |
| **Failover** | Promoting a follower to leader when the leader dies. |
| **Quorum** | The minimum number of nodes that must agree for an operation to count. |

---

## Why replicate?

| Reason | What it gives you |
|---|---|
| **High availability** | One machine dies, the others keep serving. |
| **Durability** | Data survives a disk failure because another copy exists. |
| **Read scaling** | Spread read traffic across many replicas. |
| **Lower latency** | Put a replica near users in another region. |
| **Isolation of workloads** | Run heavy analytics/reports on a replica, not the main database. |

---

## Leader-follower (primary-replica)

The most common setup. Used by default in PostgreSQL, MySQL, MongoDB replica sets, and many others.

```
                    writes
         App ─────────────────► ┌──────────┐
          │                     │  Leader  │
          │                     └────┬─────┘
          │               replication│log
          │           ┌──────────────┼──────────────┐
          │           ▼              ▼              ▼
          │     ┌──────────┐   ┌──────────┐   ┌──────────┐
          └────►│Follower 1│   │Follower 2│   │Follower 3│
         reads  └──────────┘   └──────────┘   └──────────┘
```

- **All writes go to the leader.** No write conflicts, because there's one source of truth.
- The leader sends every change to followers via a replication log.
- **Reads can go to followers** — these are called **read replicas**.

### Read replicas

Most apps read far more than they write (often 10:1 or 100:1). Read replicas let you add machines to handle reads.

```
1 leader + 5 read replicas ≈ roughly 6x the read capacity
(writes still limited to what 1 leader can handle)
```

Read replicas **don't help with write scaling**. For that, you need sharding.

---

## Synchronous vs asynchronous replication

When the leader gets a write, does it wait for followers before saying "done"?

```
SYNCHRONOUS                                ASYNCHRONOUS
App ─► Leader ─► Follower                  App ─► Leader ──► "OK" to app
          ◄── "got it"                               └──► Follower (later)
       "OK" to app
```

| | Synchronous | Asynchronous |
|---|---|---|
| **Write latency** | Slower (waits for follower) | ✅ Fast |
| **Data loss if leader dies** | ✅ None (follower has it) | ❌ Recent writes can be lost |
| **If a follower is slow/down** | ❌ Writes stall | ✅ Unaffected |
| **Replica freshness** | ✅ Up to date | Lags behind |

In practice, most systems use **async**, or **semi-synchronous**: wait for **one** follower to confirm, the rest are async. That guarantees at least two copies without making every follower a bottleneck.

---

## Replication lag and its problems

With async replication, followers are behind the leader — usually by milliseconds, but under heavy load it can be seconds or more. This creates confusing bugs.

### Problem 1: Read-your-writes

You update your profile picture, refresh the page... and the old one is still there.

```
1. User ── write "new photo" ──► Leader         ✅ saved
2. User ── read profile ────────► Follower 2   (hasn't received it yet)
3. User sees OLD photo                         ❌ "my update was lost!"
```

**Fixes:**

- Read **your own** data (your profile, your settings) from the **leader**; others' data from replicas.
- After a user writes, send *their* reads to the leader for a short window (e.g. 10 seconds).
- Track the write's log position; only read from a replica that has caught up to it.

### Problem 2: Monotonic reads (going back in time)

You refresh twice and hit two different replicas: the first one is up to date, the second one is behind. A comment appears, then disappears.

**Fix:** pin each user to the same replica (e.g. by hashing user ID).

---

## Multi-leader replication

**More than one node accepts writes**, and they replicate to each other. Typically one leader per data center.

```
     Data center: India             Data center: US
     ┌──────────┐    ◄──────►     ┌──────────┐
     │ Leader A │   replicate     │ Leader B │
     └────┬─────┘    both ways    └────┬─────┘
       followers                     followers
```

- ✅ Users write to a nearby leader → low latency.
- ✅ A whole data center can go down and the other keeps accepting writes.
- ❌ **Write conflicts**: two users edit the same record in two places at once. Who wins?

Conflict resolution options:

| Strategy | How it works | Downside |
|---|---|---|
| **Last write wins (LWW)** | Highest timestamp wins | Silently loses data; clocks drift |
| **Merge** | Combine values (e.g. union of two tag lists) | Only works for some data types |
| **CRDTs** | Data types designed to merge automatically | More complex |
| **Ask the app / user** | Store both versions, resolve later | Extra app logic |

Also used by offline-first apps (each device is a "leader") and collaborative editors like Google Docs.

---

## Leaderless replication (quorums)

**No leader at all.** The client (or a coordinator) sends writes and reads to **several replicas at once**. Used by **Cassandra**, **DynamoDB**, Riak. Inspired by Amazon's Dynamo paper.

Three numbers control it:

| Symbol | Meaning |
|---|---|
| **N** | Number of replicas holding each piece of data |
| **W** | Replicas that must confirm a **write** before it's successful |
| **R** | Replicas that must answer a **read** |

**The quorum rule:** if **W + R > N**, every read overlaps with at least one replica that has the latest write.

```
N = 3, W = 2, R = 2     (2 + 2 > 3 ✅)

Write "x=5":   Replica A ✅   Replica B ✅   Replica C ✗ (down)   → success (2 of 3)
Read x:        Replica B → 5  Replica C → 4 (stale)                → pick newest: 5 ✅
                     ▲ overlap guaranteed: at least one read replica saw the write
```

Tuning it:

| Setting | Effect |
|---|---|
| `W = N, R = 1` | Fast reads, writes fail if any replica is down |
| `W = 1, R = N` | Fast writes, slow reads |
| `W = 2, R = 2, N = 3` | Common balanced choice; tolerates 1 node down |
| `W + R ≤ N` | Faster and more available, but reads can be stale (eventual consistency) |

Stale replicas get fixed by **read repair** (fix it when a read notices it's old) and **anti-entropy** (background process comparing replicas).

---

## Comparing the three approaches

| | Leader-follower | Multi-leader | Leaderless |
|---|---|---|---|
| **Who accepts writes** | One leader | Several leaders | Any replica |
| **Write conflicts** | ✅ None | ❌ Must resolve | ❌ Must resolve (versions) |
| **Write availability** | Leader down = pause until failover | ✅ High | ✅ High |
| **Complexity** | Low | High | Medium-high |
| **Examples** | Postgres, MySQL, MongoDB | Multi-region setups, CouchDB | Cassandra, DynamoDB, Riak |

---

## Failover

When the leader dies, a follower must take over.

```
1. Detect     → leader misses heartbeats for ~N seconds → "presumed dead"
2. Elect      → pick the most up-to-date follower (via consensus, or an orchestrator)
3. Reconfigure→ clients/other followers now point to the new leader
4. Old leader → if it comes back, it must step down and become a follower
```

What can go wrong:

- **Lost writes** — with async replication, the new leader may be missing the old leader's last few writes.
- **Split brain** — the old leader wasn't dead, just slow. Now **two leaders** accept writes and data diverges. Prevented with fencing (forcibly cutting off the old leader) and consensus (Raft, Paxos, ZooKeeper, etcd).
- **Timeout tuning** — too short: failover on a brief network blip. Too long: minutes of downtime.

Managed services (AWS RDS Multi-AZ, Aurora, Cloud SQL) automate this, typically in seconds to a minute or two.
