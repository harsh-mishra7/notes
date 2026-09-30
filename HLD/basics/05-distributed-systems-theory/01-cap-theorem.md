# CAP Theorem

## TL;DR

**CAP = Consistency, Availability, Partition tolerance.**

When your data lives on more than one machine and the network between them breaks, you have to pick one:

- **Refuse to answer** until the machines can talk again (stay **consistent**), or
- **Answer anyway** with whatever data you have (stay **available**), even if it might be stale.

You can't have both *during a network split*. That's the whole theorem.

The famous "pick 2 out of 3" is misleading. Partitions aren't optional in a distributed system, so the real choice is **CP vs AP — and only while a partition is happening.**

---

## The problem: copies of data that can't talk

As soon as you have 2+ servers holding the same data (for speed or for safety), they need to stay in sync over a network. Networks fail: cables get cut, switches die, a data center loses its uplink, a GC pause makes a node look dead.

```
         ┌──────────┐        network        ┌──────────┐
 User A ─►  Node 1  │ ◄────── ✂ ✂ ✂ ──────► │  Node 2  ◄─ User B
         │ x = 5    │     (link is broken)  │ x = 5    │
         └──────────┘                       └──────────┘

 User A writes x = 10 on Node 1.
 User B reads x from Node 2.  What should Node 2 say?
```

Node 2 has exactly two options:

1. Say "sorry, I can't be sure, try later" → **consistent but not available** (CP)
2. Say "x = 5" → **available but not consistent** (AP)

There is no third option. Node 2 physically cannot know about the write.

---

## The analogy that makes it click

**Two bank branches with a phone line between them.**

You and your partner share an account with ₹1,000. The branches call each other to sync every withdrawal. One day the phone line goes dead.

You walk into Branch A and ask to withdraw ₹1,000. Your partner walks into Branch B at the same moment and asks for ₹1,000 too.

- **CP branch:** "Our line to the other branch is down. We can't confirm your balance. No withdrawals until it's fixed." You're annoyed, but the bank never loses money.
- **AP branch:** "Sure, here's ₹1,000." Both of you get cash. The account is now -₹1,000. The bank will sort it out later (overdraft fee, reconciliation).

Neither branch is "wrong". They made different trade-offs.

---

## The vocabulary (precise definitions)

| Term | Precise meaning |
|---|---|
| **Consistency (C)** | Every read returns the **most recent write** (or an error). All nodes behave like a single copy. This is *linearizability* — not the "C" in ACID. |
| **Availability (A)** | Every request to a **non-failed node** gets a **non-error response** — no matter how long the partition lasts. |
| **Partition tolerance (P)** | The system keeps operating even when the network **drops or delays messages** between nodes. |
| **Partition** | Some nodes can't reach some other nodes. Both sides are alive; they just can't talk. |

Note how strict "A" is: *every* live node must answer. A system that answers from the majority side but errors on the minority side is **not** "A" in the CAP sense.

---

## Why P is not optional

"Pick 2 of 3" suggests you could choose **CA** — consistent and available, just not partition tolerant.

In a distributed system, that doesn't mean anything. You don't get to *choose* whether the network fails. Choosing "no P" means "my system breaks in undefined ways when the network fails" — which is not a design, it's a bug.

```
 "Pick 2 of 3"  (misleading)          What's actually true

      C                               Is there a partition right now?
     / \                                  │
    /   \                        ┌── YES ─┴── NO ──┐
   A ─── P                       │                 │
                           Choose C or A     You can have C and A
                           (CP vs AP)        (but see PACELC below)
```

**CA only really exists on a single machine** (e.g. one Postgres server). No network between copies = no partition = no problem. The moment you replicate, you're choosing CP or AP.

---

## CP vs AP side by side

| | **CP** (Consistency over Availability) | **AP** (Availability over Consistency) |
|---|---|---|
| During a partition | Some requests get errors/timeouts | Every node keeps answering |
| Data you read | Always the latest | Possibly stale |
| After the partition heals | Nothing to fix | Conflicting writes must be merged |
| Good for | Money, inventory, locks, leader election | Feeds, likes, carts, DNS, sessions |
| Failure feels like | "Service unavailable, try again" | "Why is my like count wrong?" |

### Examples of CP systems

| System | Why it's CP |
|---|---|
| **ZooKeeper / etcd / Consul** | Use consensus (ZAB/Raft). The minority side of a partition refuses writes. |
| **HBase** | One region server owns each key range; if it's unreachable, that range is unavailable. |
| **MongoDB** (default, majority writes) | Only the primary takes writes; a primary cut off from the majority steps down. |
| **Google Spanner** | Strongly consistent; chooses C and relies on a very reliable network to make partitions rare. |
| **Traditional RDBMS with sync replication** | A write waits for the replica; if the replica is unreachable, writes block. |

### Examples of AP systems

| System | Why it's AP |
|---|---|
| **Cassandra** (typical config) | Any replica accepts reads/writes; conflicts resolved later (last-write-wins). |
| **DynamoDB** (default eventual reads) | Designed from the Dynamo paper: always-writable shopping carts. |
| **Riak, CouchDB** | Multi-master; accept writes on both sides, reconcile after. |
| **DNS** | Serves cached records even if they're outdated. |

**Important:** many databases are *tunable*. Cassandra with `QUORUM` reads and writes behaves much more like CP. So it's more accurate to say "this *configuration* is CP" than "this *database* is CP".

---

## Common misconceptions

**1. "Pick any 2 of 3."**
No. P is forced on you. You pick C or A *when a partition happens*.

**2. "AP means no consistency at all."**
No. AP systems are usually *eventually* consistent: once the partition heals, replicas converge. You just give up the guarantee *during* the split.

**3. "CP means the system is down during a partition."**
Not entirely. Usually the **majority side keeps working**; only the minority side (or requests that need it) fail.

**4. "The C in CAP is the C in ACID."**
No. ACID's C means "the database respects your rules/constraints". CAP's C means linearizability — every read sees the latest write.

**5. "My system is CP, so it's always consistent and slow" / "AP, so always fast."**
CAP says nothing about normal operation or latency. That's what PACELC is for.

**6. "Partitions are rare, so CAP doesn't matter."**
Partitions include slow networks, long GC pauses, and overloaded nodes. From the outside, "slow" and "dead" look identical. They happen more than you think.

---

## PACELC: what about when things are fine?

CAP only talks about the partition case. But most of the time there *is* no partition — and there's still a trade-off.

**PACELC:**

> If there's a **P**artition → choose **A** or **C**.
> **E**lse (normal operation) → choose **L**atency or **C**onsistency.

Why the "else" trade-off? To be strongly consistent, a write must reach other replicas (often in other regions) *before* you say "done". That's network round trips = latency. To be fast, you reply immediately and replicate in the background = possibly stale reads.

```
        Partition?
         │
   ┌─YES─┴──NO──┐
   │            │
 A or C      L or C
             (latency vs consistency)
```

| System | During Partition | Else | Label |
|---|---|---|---|
| Cassandra, DynamoDB (default) | A | L | **PA/EL** |
| MongoDB | C | C (mostly) | **PC/EC** |
| Spanner, etcd, ZooKeeper | C | C | **PC/EC** |
| Yahoo PNUTS | C | L | **PC/EL** |
| Classic Dynamo, Riak | A | L | **PA/EL** |

PACELC is more useful day-to-day, because you pay the latency cost on *every* request, not just during rare outages.

---

## Where this shows up in HLD

- **Every "which database?" question.** Interviewers want to hear you pick a side *for this use case*: "Payments need CP; the news feed can be AP."
- **Splitting one system into parts.** Real designs mix both: order placement is CP, product reviews and view counts are AP.
- **Multi-region designs.** Cross-region writes add 100+ ms. You'll trade latency vs consistency (PACELC) constantly.
- **Explaining failure behavior.** "If region X is cut off, users there can still browse (AP) but checkout returns an error (CP)." That sentence scores points.
- **Coordination services.** Leader election, distributed locks, config — always CP (etcd, ZooKeeper), because two leaders is worse than zero.

---

## Key takeaways

- CAP is about what happens **during a network partition**: stay consistent (refuse some requests) or stay available (maybe serve stale data).
- **P is not optional** in a distributed system, so the real choice is **CP vs AP**. "CA" only exists on a single node.
- CAP's **C = linearizability**, and **A = every live node answers** — both are stricter than everyday usage.
- Many databases are **tunable**; the configuration decides, not the brand name.
- **PACELC** adds the everyday trade-off: even without a partition, you choose **latency vs consistency**.
- In interviews, choose per feature: money and locks → CP; counters, feeds, carts → AP.
