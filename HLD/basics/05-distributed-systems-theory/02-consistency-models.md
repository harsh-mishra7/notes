# Consistency Models

## Brief

**A consistency model is a promise about what a read can return after a write.**

When data is copied across several servers, a read might hit a replica that hasn't heard about the latest write yet. A consistency model tells you exactly how "behind" that read is allowed to be.

- **Strong** = every read sees the latest write, as if there were one copy.
- **Eventual** = reads might be stale, but if writes stop, all copies will eventually agree.
- Everything else sits in between, giving specific, useful guarantees (like "you'll always see your *own* writes").

Stronger = easier to reason about, but slower and less available. Weaker = faster and more available, but your app has to tolerate surprises.

---

## The problem: replicas lag behind

```
            write x = 10
 Client ─────────────────► Leader (x = 10)
                              │
                 replication  │  takes a few ms ... or seconds
                              ▼
                         Replica (x = 5)  ◄──── read x ──── Another client
                                                  gets 5 !
```

Replication takes time. During that window, different replicas hold different values. The question is: **what are readers allowed to see?**

---

## The analogy that makes it click

**A company announcement spreading through an office.**

The CEO changes the lunch policy.

- **Strong consistency:** The CEO won't say "done" until *every* employee has been told. Anyone you ask gives the new policy. But the announcement takes ages.
- **Eventual consistency:** The CEO tells a few people and walks off. The news spreads by word of mouth. Ask someone right now and they might give the old policy — but by tomorrow, everyone knows.
- **Read-your-writes:** If *you* made the announcement, you'll never be told the old policy by anyone you ask.
- **Causal:** If Alice says "the policy changed" and Bob replies "great, finally!", nobody hears Bob's reply before Alice's news.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Replica** | A copy of the data on another node. |
| **Stale read** | A read that returns an older value than the latest write. |
| **Replication lag** | How far behind a replica is. |
| **Session** | One client's sequence of requests (e.g. one logged-in user). |
| **Quorum** | A minimum number of replicas that must agree for an operation to succeed. |

---

## The models, strongest to weakest

```
STRONGER  ▲   Linearizable (strong)
(slower,  │   Sequential
 safer)   │   Causal
          │   Session guarantees: read-your-writes, monotonic reads
WEAKER    │   Eventual
(faster)  ▼
```

### 1. Strong consistency / Linearizability

Once a write completes, **every** read after it (from anyone, anywhere) returns that value or a newer one. The system behaves as if there's **one single copy** and operations happen instantly at some point in time.

```
Alice: write(x=10) ──done──┐
                           │  after this moment...
Bob:                       └── read(x) → 10   ✅ always
Carol:                          read(x) → 10  ✅ always
```

- **Cost:** writes must reach a majority (or all) replicas before returning. Higher latency; unavailable during partitions (CP).
- **Used by:** etcd, ZooKeeper, Spanner, a single-leader DB reading from the leader.

### 2. Sequential consistency (briefly)

All clients see operations in the **same order**, and each client's own operations appear in the order it issued them. But that order doesn't have to match real-time clock order.

Difference from linearizable: Bob might not see Alice's write *immediately* after it finishes — but everyone agrees on the same history. Rarely a design choice in HLD interviews; know it exists.

### 3. Causal consistency

If one operation **could have caused** another, everyone sees them in that order. Unrelated operations can appear in any order.

```
Alice posts:   "I lost my job"           (A)
Bob replies:   "So sorry to hear that"   (B, caused by A)

Causal guarantees nobody sees B without A.  ✅
Without it, Carol might see Bob's reply first — confusing.  ❌
```

- Much cheaper than strong: only *related* events need ordering.
- **Used for:** comments, chat threads, collaborative apps.

### 4. Read-your-writes (read-my-writes)

After **you** write something, **you** will always see it. Other users might still see the old value for a while.

```
You:   update profile photo ──► refresh page ──► see NEW photo  ✅
Friend:                         view your page ──► might see OLD photo (ok)
```

- Without it: you change your name, refresh, and it's the old name. Users think the save failed and click again.
- **How it's done:** read your own data from the leader, or route your session to a replica that has your write, or track a "last write version" and wait until the replica catches up.

### 5. Monotonic reads

Once you've seen a value, you'll **never see an older one** afterwards. Time doesn't go backwards for you.

```
Without monotonic reads:
  refresh 1 → hits Replica A (up to date)  → 12 comments
  refresh 2 → hits Replica B (lagging)     → 9 comments   ❌ comments "disappeared"

With it: stick each user to one replica (e.g. hash user_id → replica).
```

### 6. Eventual consistency

If no new writes happen, **all replicas will eventually converge** to the same value. No promise about *when*, and no promise about what you see in the meantime.

- **Cost:** almost nothing. Fast writes, highly available (AP).
- **Catch:** concurrent writes can conflict. The system needs a rule: last-write-wins (by timestamp), version vectors, or CRDTs (data types that merge automatically).
- **Used by:** DNS, Cassandra/DynamoDB defaults, like counters, view counts, caches.

---

## Two examples: bank balance vs likes count

| | **Bank balance** | **Likes on a post** |
|---|---|---|
| What happens if a read is stale? | Someone withdraws money that isn't there. Real loss. | Shows 1,203 instead of 1,205. Nobody cares. |
| Concurrent writes | Must be ordered exactly (no double spending) | Just add them up eventually |
| Right model | **Strong** | **Eventual** |
| Trade-off accepted | Slower, may reject requests during failures | Tiny inaccuracy for speed and uptime |

Most real apps are a **mix**: the payment service is strongly consistent, the feed is eventual, the profile page is read-your-writes.

---

## Quorums: tuning consistency with numbers

Many leaderless databases (Cassandra, DynamoDB, Riak) let you pick consistency per request using three numbers:

| Symbol | Meaning |
|---|---|
| **N** | Number of replicas holding each piece of data |
| **W** | Replicas that must confirm a **write** before it's "done" |
| **R** | Replicas that must answer a **read** |

**The rule: if R + W > N, every read overlaps with the latest write.**

```
N = 3 replicas, W = 2, R = 2    (2 + 2 = 4 > 3)

Write x=10 ──► [A ✅] [B ✅] [C  ✗ didn't get it yet]
Read x     ──► [   ] [B ✅] [C  ]   ← any 2 you pick include at least one
                                       node that has x=10
```

Because the write set and read set **must share at least one node**, the read sees the new value (the client picks the highest version among responses).

| Config (N=3) | R + W | Behavior |
|---|---|---|
| W=3, R=1 | 4 > 3 | Fast reads, slow writes; one dead node blocks writes |
| W=1, R=3 | 4 > 3 | Fast writes, slow reads |
| W=2, R=2 | 4 > 3 | Balanced — the classic `QUORUM` setting |
| W=1, R=1 | 2 ≤ 3 | Fastest, but reads can be stale — eventual |

**Caveat:** R + W > N gives "reads see the latest *completed* write" in the normal case, but it's not full linearizability on its own (sloppy quorums, concurrent writes, and failed partial writes create edge cases). It's still the standard interview answer for "how do you make reads fresh in a leaderless DB?"

---

## Which model fits which use case

| Use case | Model | Why |
|---|---|---|
| Bank balance, payments | Strong | Money can't be double-spent |
| Inventory for last few items / seat booking | Strong | Overselling is a real cost |
| Distributed locks, leader election | Strong | Two leaders = corruption |
| Usernames / unique constraints | Strong | Two people can't claim the same name |
| Chat messages, comment threads | Causal | Replies must follow what they reply to |
| User editing their own profile/settings | Read-your-writes | User must see their own change |
| Scrolling a feed, comment counts | Monotonic reads | Things shouldn't "un-happen" on refresh |
| Likes, views, follower counts | Eventual | Exact number doesn't matter right now |
| Product catalog, search index | Eventual | Seconds of lag is fine |
| DNS, CDN caches | Eventual | Speed and uptime matter more |
