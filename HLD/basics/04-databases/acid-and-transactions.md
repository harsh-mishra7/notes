# ACID and Transactions

## TL;DR

A **transaction** is a group of database operations that should be treated as **one unit**: either all of them happen, or none of them do.

**ACID** is the set of four guarantees a transaction gives you:

- **A**tomicity — all or nothing.
- **C**onsistency — the data always follows your rules.
- **I**solation — concurrent transactions don't mess each other up.
- **D**urability — once committed, it survives crashes.

**Isolation levels** let you trade some correctness for speed. **BASE** is the looser alternative many distributed NoSQL systems choose.

---

## The analogy that makes it click

**A bank transfer.**

Asha sends ₹500 to Ravi. That's two steps:

```
1. Asha's balance  -= 500
2. Ravi's balance  += 500
```

Now imagine the server crashes between step 1 and step 2. ₹500 has vanished from the universe. Asha is angry, Ravi got nothing, and the bank's books don't balance.

A transaction wraps both steps so the bank can promise: **either the full transfer happened, or nothing happened.** No in-between state is ever visible or permanent.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 'asha';
  UPDATE accounts SET balance = balance + 500 WHERE id = 'ravi';
COMMIT;   -- or ROLLBACK if anything fails
```

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Transaction** | A group of operations treated as one unit. |
| **Commit** | "Make it permanent." All changes become visible and durable. |
| **Rollback / abort** | "Undo everything since BEGIN." |
| **Concurrency** | Many transactions running at the same time. |
| **Anomaly** | A weird result caused by transactions interfering with each other. |
| **WAL (write-ahead log)** | A log written to disk *before* changes are applied, used for recovery. |

---

## A — Atomicity

**All or nothing.** If any step fails, the whole transaction is rolled back as if it never started.

```
BEGIN
  Asha -500      ✅
  Ravi +500      💥 crash / error
ROLLBACK  ──►  Asha's -500 is undone. Balances back to the start.
```

"Atomic" here means *indivisible*, not "about concurrency". It's about failure handling.

How it's done: the database logs what it's changing (undo/redo logs), so it can reverse partial work.

---

## C — Consistency

**The data always moves from one valid state to another valid state.** Your rules (constraints) are never broken after a commit.

Rules like:

- Balance can't go negative (`CHECK (balance >= 0)`).
- Every order must point to a real user (foreign key).
- Email must be unique.

If Asha only has ₹300, the transfer of ₹500 violates the rule, so the whole transaction is rejected.

> Note: Consistency is partly the **application's** job. The database enforces the constraints you declare; it can't know business rules you never told it. Also, this "C" is **not** the same "C" as in the CAP theorem (that one is about replicas agreeing).

---

## I — Isolation

**Concurrent transactions behave as if they ran one after another.** One transaction shouldn't see another's half-finished work.

Without isolation, two transfers happening at the same time can go wrong:

```
Asha's balance = 1000

Transaction T1 (send 500 to Ravi)     Transaction T2 (send 800 to Meera)
read balance → 1000
                                      read balance → 1000
check 1000 >= 500 ✅
                                      check 1000 >= 800 ✅
write balance = 500
                                      write balance = 200
COMMIT                                COMMIT

Final balance: 200.  But 1300 was sent from 1000.  ❌  (a "lost update")
```

Full isolation is expensive, so databases offer **isolation levels** (below).

---

## D — Durability

**Once the database says "committed", the data is safe** — even if the power goes out one millisecond later.

How it's done:

- **Write-ahead log (WAL)**: changes are written to a log on disk and flushed (`fsync`) *before* the commit is acknowledged. After a crash, the database replays the log.
- In distributed systems, durability often also means **copied to other replicas** before acknowledging.

---

## Isolation levels and anomalies

First, the anomalies (the bugs isolation prevents):

| Anomaly | What happens | Example |
|---|---|---|
| **Dirty read** | You read data another transaction wrote but **hasn't committed** (and might roll back). | T2 sees Asha's balance as 500 mid-transfer; T1 then rolls back. T2 acted on data that never existed. |
| **Non-repeatable read** | You read the **same row twice** and get **different values**, because someone committed a change in between. | Read balance = 1000. Someone commits a withdrawal. Read again = 400. |
| **Phantom read** | You run the **same query twice** and get a **different set of rows**, because someone inserted/deleted matching rows. | `COUNT(*) WHERE status='pending'` = 5, then 6 in the same transaction. |

Now the four standard SQL isolation levels, from weakest to strongest:

| Isolation level | Dirty read | Non-repeatable read | Phantom read | Speed |
|---|---|---|---|---|
| **Read Uncommitted** | ❌ possible | ❌ possible | ❌ possible | Fastest |
| **Read Committed** | ✅ prevented | ❌ possible | ❌ possible | Fast |
| **Repeatable Read** | ✅ prevented | ✅ prevented | ❌ possible* | Medium |
| **Serializable** | ✅ prevented | ✅ prevented | ✅ prevented | Slowest |

\* In practice, PostgreSQL's and MySQL InnoDB's Repeatable Read (via snapshots and, in MySQL, gap locks) prevent most phantoms too. The table shows what the SQL standard guarantees.

In plain English:

- **Read Uncommitted** — you might see other people's drafts. Almost never used.
- **Read Committed** — you only see committed data, but it can change under you. **Default in PostgreSQL, Oracle, SQL Server.**
- **Repeatable Read** — rows you've read stay the same for your whole transaction (you work on a snapshot). **Default in MySQL InnoDB.**
- **Serializable** — the result is as if transactions ran one at a time. Safest, but more blocking/retries.

```
Weaker ◄──────────────────────────────────────────────► Stronger
Read Uncommitted → Read Committed → Repeatable Read → Serializable
more speed, more anomalies                    fewer anomalies, more cost
```

**Rule of thumb:** use the default (Read Committed) for most work. For money, inventory, or bookings, use Serializable, or explicit row locks (`SELECT ... FOR UPDATE`) on the rows you're about to change.

---

## BASE: the alternative

Many distributed NoSQL databases (Cassandra, DynamoDB by default, Riak) choose availability and scale over strict ACID. Their philosophy is called **BASE**:

| Letter | Meaning |
|---|---|
| **BA** — Basically Available | The system keeps answering, even during failures. |
| **S** — Soft state | Data may be changing in the background as replicas sync. |
| **E** — Eventually consistent | If writes stop, all replicas will *eventually* agree. |

---

## ACID vs BASE

| | ACID | BASE |
|---|---|---|
| **Priority** | Correctness | Availability and scale |
| **After a write** | Everyone sees the new value | Some readers may briefly see the old value |
| **Failure behavior** | May reject/block requests to stay correct | Keeps serving, reconciles later |
| **Scaling** | Harder across many machines | Built for many machines |
| **Typical systems** | PostgreSQL, MySQL, Spanner | Cassandra, DynamoDB (default), Riak |
| **Good for** | Payments, orders, inventory, bookings | Likes, view counts, feeds, logs, analytics |

```
"Did the payment go through?"   → must be ACID. Wrong = lost money.
"How many likes on this post?"  → BASE is fine. 1,203 vs 1,204 for a second? Nobody cares.
```

It's not all-or-nothing: many systems use ACID for the core (orders, payments) and BASE for the rest (feeds, counters).

---

## Where this shows up in HLD

- Payments, wallets, ticket booking, inventory ("don't sell the last seat twice") — interviewers expect you to say **transactions** and usually **row locking or serializable isolation**.
- When data is **sharded across machines**, a transaction touching two shards becomes a **distributed transaction** (e.g. two-phase commit) — slow and complex. Designs often avoid it with **sagas** (a chain of local transactions with compensating "undo" steps).
- Choosing Cassandra/DynamoDB means accepting BASE-style trade-offs — say so explicitly.
- "Strong vs eventual consistency" questions tie directly to ACID vs BASE and to replication.

---

## Key takeaways

- A transaction = all-or-nothing group of operations. ACID is its four guarantees.
- Atomicity handles failures, Consistency enforces rules, Isolation handles concurrency, Durability survives crashes.
- Isolation levels trade anomalies (dirty, non-repeatable, phantom reads) for speed.
- Read Committed is a common default; use Serializable or row locks for money and inventory.
- BASE = availability + eventual consistency; fine for counters and feeds, not for payments.
