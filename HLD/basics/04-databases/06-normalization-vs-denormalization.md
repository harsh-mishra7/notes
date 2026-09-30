# Normalization vs Denormalization

## TL;DR

**Normalization** = store each fact **exactly once**, and link to it from everywhere else. Less duplication, fewer bugs when data changes, but reads need **joins**.

**Denormalization** = deliberately **copy data** into the places it's read, so reads are fast and simple. Faster reads, but writes must update every copy.

Normalize by default. Denormalize **on purpose**, for specific hot read paths.

---

## The analogy that makes it click

**Your friend changes their phone number.**

- **Normalized:** you keep their number in *one* contact entry. Every chat, group, and calendar invite points to that contact. Change it once, it's correct everywhere.
- **Denormalized:** you wrote their number on 12 sticky notes around the house. Finding it is instant — there's one right next to you. But when it changes, you must find and update all 12. Miss one, and you're calling a dead number.

Normalization optimizes for **changing** data. Denormalization optimizes for **reading** it.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Redundancy** | The same fact stored in more than one place. |
| **Anomaly** | A bug caused by redundancy (data getting out of sync). |
| **Normal form (1NF, 2NF, 3NF)** | Increasingly strict rules for removing redundancy. |
| **Primary key** | The column(s) that uniquely identify a row. |
| **Join** | Recombining data split across tables at read time. |
| **Materialized view** | A precomputed, stored query result — a managed form of denormalization. |

---

## The problem: one messy table

Here's an orders table where everything is crammed together:

```
orders_messy
┌──────────┬──────────┬──────────────┬─────────────┬──────────────────┬──────────┐
│ order_id │ customer │ cust_email   │ cust_city   │ products         │ total    │
├──────────┼──────────┼──────────────┼─────────────┼──────────────────┼──────────┤
│ 101      │ Asha     │ asha@x.com   │ Pune        │ "Pen, Notebook"  │ 150      │
│ 102      │ Asha     │ asha@x.com   │ Pune        │ "Bag"            │ 900      │
│ 103      │ Ravi     │ ravi@x.com   │ Delhi       │ "Pen"            │ 20       │
└──────────┴──────────┴──────────────┴─────────────┴──────────────────┴──────────┘
```

Asha's email and city are stored twice. The products column is a list crammed into a string.

---

## The three anomalies redundancy causes

| Anomaly | What goes wrong | In the table above |
|---|---|---|
| **Update anomaly** | You change a fact in one row but not another → contradictions. | Asha moves to Mumbai; you update order 101 but not 102. Where does she live now? |
| **Insert anomaly** | You can't store a fact without some unrelated fact. | Can't add a new customer until they place an order. |
| **Delete anomaly** | Deleting one thing accidentally deletes another fact. | Delete Ravi's only order → you've lost that Ravi exists at all. |

Normalization is the process of removing these.

---

## Normal forms, the practical version

You mostly need to know the first three. Each builds on the previous.

### 1NF — One value per cell

**Rule:** every cell holds a single, atomic value. No lists, no repeating columns (`product1`, `product2`, `product3`).

```
❌  products = "Pen, Notebook"

✅  order_items
    ┌──────────┬───────────┐
    │ order_id │ product   │
    ├──────────┼───────────┤
    │ 101      │ Pen       │
    │ 101      │ Notebook  │
    │ 102      │ Bag       │
    └──────────┴───────────┘
```

Now you can query "all orders containing a Pen" without string parsing.

### 2NF — No partial dependencies

**Rule:** 1NF, and every non-key column depends on the **whole** primary key, not just part of it. (Only matters when the key is made of multiple columns.)

```
order_items (key = order_id + product_id)
┌──────────┬────────────┬──────────┬──────────────┐
│ order_id │ product_id │ quantity │ product_name │
└──────────┴────────────┴──────────┴──────────────┘
                              ▲            ▲
                  depends on both   depends only on product_id  ❌
```

`product_name` depends only on `product_id`, so move it to its own `products` table.

### 3NF — No transitive dependencies

**Rule:** 2NF, and non-key columns depend **only on the key**, not on other non-key columns.

```
orders (key = order_id)
┌──────────┬─────────────┬────────────┬───────────┐
│ order_id │ customer_id │ cust_email │ cust_city │
└──────────┴─────────────┴────────────┴───────────┘
                               ▲             ▲
            depend on customer_id, not on order_id  ❌
```

`cust_email` and `cust_city` are facts about the **customer**, not the order. Move them to a `customers` table.

> A classic memory aid: every non-key column must depend on "**the key, the whole key, and nothing but the key**."

### The normalized result

```
customers                 orders                       order_items               products
┌────┬──────┬───────┐    ┌─────┬─────────┬───────┐    ┌──────────┬────────┬───┐  ┌────┬──────────┬───────┐
│ id │ name │ city  │◄───┤ id  │ cust_id │ date  │◄───┤ order_id │ prod_id│qty│─►│ id │ name     │ price │
└────┴──────┴───────┘    └─────┴─────────┴───────┘    └──────────┴────────┴───┘  └────┴──────────┴───────┘
```

Every fact lives in one place. Asha moves? Update one row.

There are higher forms (BCNF, 4NF, 5NF), but in practice **3NF is the usual target**.

---

## Why denormalize?

Normalized data is correct and tidy, but **reading it means joining**. At scale, joins get expensive:

- Joining 4-5 large tables on every page load burns CPU and I/O.
- Once data is **sharded**, the tables you want to join may be on **different machines** — joins become slow or impossible.
- Many NoSQL databases **don't support joins at all**.

So for hot read paths, you **precompute the answer** and store it in the shape it's read.

```
NORMALIZED READ                              DENORMALIZED READ
customers ─┐                                 order_summary (one row, already combined)
orders ────┼─ JOIN ─► result   (4 tables)    ────────────────────────► result  ✅
order_items┤                                  one lookup
products ──┘
```

Common ways to denormalize:

- **Copy a field** — store `customer_name` directly on each order.
- **Precomputed counters** — store `like_count` on a post instead of `COUNT(*)` every time.
- **Embedded documents** — in MongoDB, keep a post's comments inside the post document.
- **Materialized views** — let the database store and refresh a query result.
- **Read-optimized tables** — one table per query pattern (the standard Cassandra approach).

---

## Example 1: E-commerce order

**Price at time of purchase.** If an order just points to `products.price`, then raising a price tomorrow silently changes the total of every past order. Wrong!

So orders **copy** the price (and product name) at purchase time:

```
order_items
┌──────────┬────────────┬──────────────┬──────────────────┬─────┐
│ order_id │ product_id │ product_name │ price_at_purchase│ qty │
└──────────┴────────────┴──────────────┴──────────────────┴─────┘
                               ▲ copied on purpose ▲
```

This is denormalization that's actually *more correct* — it's a snapshot of a historical fact, not a duplicate of a current one. Same with a shipping address copied onto an order.

---

## Example 2: A news feed

**Normalized:** to build your feed, find everyone you follow, fetch their recent posts, fetch each author's name and avatar, count likes, sort by time.

```sql
SELECT p.*, u.name, u.avatar, COUNT(l.id) AS likes
FROM follows f
JOIN posts p  ON p.author_id = f.followee_id
JOIN users u  ON u.id = p.author_id
LEFT JOIN likes l ON l.post_id = p.id
WHERE f.follower_id = :me
GROUP BY p.id, u.name, u.avatar
ORDER BY p.created_at DESC
LIMIT 50;
```

For hundreds of millions of users refreshing constantly, that's far too slow.

**Denormalized:** when someone posts, **push** a ready-to-render entry into each follower's feed (fan-out on write):

```
feed_by_user  (partition key: user_id, sorted by time)
┌─────────┬────────────┬─────────┬─────────────┬──────────────┬────────────┐
│ user_id │ created_at │ post_id │ author_name │ author_avatar│ text_prev  │
└─────────┴────────────┴─────────┴─────────────┴──────────────┴────────────┘

Read your feed = one partition read. ✅
```

The cost: a post by someone with 10,000 followers = 10,000 writes. And if an author changes their name, old feed entries show the old one (often acceptable, or fixed by storing only `author_id` and looking up names from a cache).

---

## The trade-offs

| | Normalized | Denormalized |
|---|---|---|
| **Duplication** | ✅ Minimal | ❌ Data copied in many places |
| **Reads** | Slower, need joins | ✅ Fast, often one lookup |
| **Writes** | ✅ Simple, one place | ❌ Must update every copy |
| **Consistency** | ✅ One source of truth | ❌ Copies can drift out of sync |
| **Storage** | ✅ Smaller | ❌ Bigger |
| **Flexibility for new queries** | ✅ Join however you like | ❌ Designed for specific queries |
| **Fits** | OLTP systems, write-heavy, correctness-critical | Read-heavy, large scale, NoSQL, analytics |

**Rule of thumb:** start normalized (3NF). When a specific read is too slow and you've tried indexes and caching, denormalize **that one path**, and decide how copies stay in sync (same transaction, async events/CDC, or accept some staleness).

---

## Where this shows up in HLD

- News feeds, timelines, and dashboards are classic denormalization questions: "fan-out on write vs fan-out on read".
- Choosing Cassandra or DynamoDB **forces** denormalization: you model one table per query, because there are no joins.
- Sharded SQL systems denormalize to avoid cross-shard joins.
- Interviewers look for you to name the **sync mechanism**: how do copies get updated, and how stale can they be?
- Counters (likes, views, followers) are almost always stored denormalized, often in a cache.

---

## Key takeaways

- Normalization stores each fact once, preventing update, insert, and delete anomalies.
- 1NF: atomic values. 2NF: depend on the whole key. 3NF: depend only on the key.
- Denormalization copies data to make reads fast, at the cost of heavier writes and possible inconsistency.
- Some copies are correct by design, like price at time of purchase.
- Start normalized; denormalize specific hot paths deliberately, with a clear plan to keep copies in sync.
