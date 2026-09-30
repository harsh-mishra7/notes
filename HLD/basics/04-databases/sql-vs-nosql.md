# SQL vs NoSQL

## TL;DR

**SQL databases** store data in **tables with a fixed structure** (rows and columns) and let you connect tables together with joins. Think PostgreSQL, MySQL.

**NoSQL databases** is a catch-all name for "everything that isn't a classic table database". They trade some of SQL's structure and guarantees for **flexibility and easy horizontal scaling**. There are four main families: document, key-value, wide-column, and graph.

Neither is "better". SQL is the safe default. You pick NoSQL when your data shape or scale makes a table database painful.

---

## The analogy that makes it click

**SQL is a well-organized filing cabinet with printed forms.**

Every customer gets the same form: name, email, phone. Every order gets its own form, with a "customer ID" box that points back to the customer's form. It's strict (you can't just scribble a new field in the margin), but you can always cross-reference everything reliably.

**NoSQL is a set of specialized storage tools.**

- A **document** store is a folder per customer where you stuff everything about them, in any shape.
- A **key-value** store is a coat check: hand over a ticket, get your coat back. Nothing else.
- A **wide-column** store is a giant ledger that's easy to keep appending to, split across many books.
- A **graph** store is a corkboard with pins and strings showing who's connected to whom.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Schema** | The declared structure of your data (which fields, which types). |
| **Relational** | Data split into tables that reference each other by keys. |
| **Join** | Combining rows from two tables using a shared key. |
| **Horizontal scaling** | Adding more machines, instead of buying one bigger machine (vertical scaling). |
| **Schema-on-write** | Structure is checked when data is saved (SQL). |
| **Schema-on-read** | Structure is interpreted when data is read (most NoSQL). |

---

## The relational model (SQL)

Data lives in tables. Each table has a fixed set of columns. Rows reference other rows using **foreign keys**.

```
users                          orders
┌────┬───────┬──────────────┐  ┌─────┬─────────┬────────┐
│ id │ name  │ email        │  │ id  │ user_id │ total  │
├────┼───────┼──────────────┤  ├─────┼─────────┼────────┤
│ 1  │ Asha  │ asha@x.com   │◄─┤ 101 │ 1       │ 499    │
│ 2  │ Ravi  │ ravi@x.com   │  │ 102 │ 1       │ 120    │
└────┴───────┴──────────────┘  │ 103 │ 2       │ 999    │
                               └─────┴─────────┴────────┘
```

You ask questions with **SQL**, a declarative language: you say *what* you want, the database figures out *how*.

```sql
SELECT u.name, SUM(o.total)
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.name;
```

What SQL databases are great at:

- **Joins** — ask questions across many tables you didn't plan for in advance.
- **Transactions (ACID)** — "move money from A to B" either fully happens or doesn't happen at all.
- **Constraints** — the database itself refuses bad data (duplicate email, order for a user that doesn't exist).
- **Maturity** — decades of tooling, knowledge, and battle-testing.

Where they struggle:

- **Scaling writes across many machines** is hard. Joins and transactions assume the data is all in one place.
- **Changing the schema** on a huge table (e.g. adding a column to 2 billion rows) can be slow and risky.
- **Uneven or nested data** (every product has different attributes) fits awkwardly into fixed columns.

Examples: PostgreSQL, MySQL, SQL Server, Oracle, SQLite.

---

## The four NoSQL families

### 1. Document stores

Each record is a **document** (usually JSON-like). Related data is kept *inside* the document instead of in separate tables.

```json
{
  "_id": "user_1",
  "name": "Asha",
  "addresses": [
    { "city": "Pune", "pin": "411001" }
  ],
  "preferences": { "theme": "dark" }
}
```

- Different documents can have different fields.
- Reading one user = one lookup, no joins.
- Good for: product catalogs, user profiles, CMS content, anything with nested/varying shape.
- Examples: **MongoDB**, Couchbase, Firestore.

### 2. Key-value stores

The simplest model: a giant **dictionary / hash map**. You give a key, you get a value. The database doesn't care what's inside the value.

```
"session:abc123"   →  { userId: 1, expires: ... }
"cart:user_1"      →  [ item_9, item_42 ]
"rate:ip:1.2.3.4"  →  17
```

- Extremely fast, extremely easy to scale.
- You can't really query "all values where X" — you need the key.
- Good for: caching, sessions, shopping carts, rate limiters, feature flags.
- Examples: **Redis** (in-memory, often a cache), **DynamoDB** (durable, managed; also supports some document-style features), Memcached.

### 3. Wide-column stores

Data is grouped by a **partition key** and sorted within that partition by a **clustering key**. Rows in the same table can have different columns. Built for **massive write volume** spread across many machines.

```
partition key: sensor_id     clustering key: timestamp (sorted)

sensor_42 │ 10:00:00 → temp=21.3 │ 10:00:05 → temp=21.4 │ 10:00:10 → temp=21.6 │ ...
sensor_77 │ 10:00:00 → temp=18.9 │ 10:00:05 → temp=19.0 │ ...
```

- Writes are very cheap (appends, see LSM trees in `indexing.md`).
- You design tables around your **queries**, not around your entities. One query pattern often = one table.
- Good for: time-series, event logs, messaging history, IoT, activity feeds at huge scale.
- Examples: **Cassandra**, ScyllaDB, HBase, Google Bigtable.

### 4. Graph databases

Data is stored as **nodes** (things) and **edges** (relationships). Relationships are first-class, so walking them is fast.

```
   (Asha) ──FRIENDS_WITH──► (Ravi) ──FRIENDS_WITH──► (Meera)
     │                                                  │
     └──────LIKES──► (Post #7) ◄────────LIKES───────────┘
```

- "Friends of friends who like the same post" is a simple query here, but a nightmare of self-joins in SQL.
- Good for: social networks, recommendations, fraud detection (rings of connected accounts), knowledge graphs.
- Examples: **Neo4j**, Amazon Neptune, JanusGraph.

---

## Side-by-side comparison

| | SQL (relational) | Document | Key-value | Wide-column | Graph |
|---|---|---|---|---|---|
| **Data shape** | Tables, fixed columns | JSON documents | Key → blob | Partitions of sorted rows | Nodes + edges |
| **Schema** | Strict | Flexible | None | Flexible per row | Flexible |
| **Joins** | ✅ Excellent | ⚠️ Limited | ❌ None | ❌ None | ✅ Via traversal |
| **Transactions** | ✅ Full ACID | ✅ Often (per-doc, some multi-doc) | ⚠️ Limited | ⚠️ Limited | ✅ Often |
| **Horizontal scaling** | ⚠️ Harder | ✅ Built-in | ✅ Easiest | ✅ Built for it | ⚠️ Harder |
| **Query flexibility** | ✅ Very high | ✅ Good | ❌ Key only | ⚠️ Must match table design | ✅ For relationships |
| **Typical use** | Orders, payments, users | Catalogs, profiles | Cache, sessions | Logs, time-series, chat | Social, fraud |

> Modern lines are blurry: PostgreSQL has a JSON column type, MongoDB has multi-document transactions, and "NewSQL" systems (CockroachDB, Spanner, YugabyteDB) give SQL + ACID with horizontal scaling. The families above are still the right mental model.

---

## Decision guide: which one do I pick?

```
Start here
   │
   ├─ Do you need strong transactions across multiple records?
   │  (money, inventory, bookings)
   │        └─ YES ──► SQL
   │
   ├─ Is it just "look up X by its ID, super fast"?
   │  (sessions, cache, counters)
   │        └─ YES ──► Key-value (Redis / DynamoDB)
   │
   ├─ Does each record have a nested, varying shape that you read as a whole?
   │  (product with different attributes per category)
   │        └─ YES ──► Document (MongoDB)
   │
   ├─ Huge write volume, append-heavy, queried by a known key + time range?
   │  (metrics, logs, chat messages)
   │        └─ YES ──► Wide-column (Cassandra)
   │
   ├─ Are the relationships themselves the main thing you query?
   │  (friends-of-friends, shortest path)
   │        └─ YES ──► Graph (Neo4j)
   │
   └─ Not sure? ──► SQL. It's the safest default and scales further than people think.
```

**Rule of thumb:** SQL when your data is *related and must be correct*. NoSQL when your data is *simple to access but huge*, or has a *shape that doesn't fit tables*.

Real systems usually use **more than one** (this is called *polyglot persistence*):

```
E-commerce app
 ├─ PostgreSQL   → users, orders, payments   (needs ACID)
 ├─ Redis        → sessions, cart, cache     (needs speed)
 ├─ Elasticsearch→ product search            (needs full-text search)
 └─ Cassandra    → clickstream / activity    (needs write throughput)
```

---

## Common myths

| Myth | Reality |
|---|---|
| "NoSQL is faster" | Faster for the access pattern it's designed for. Slower (or impossible) for others. |
| "SQL can't scale" | It scales far with replicas, caching, and sharding. Many huge companies run on MySQL/Postgres. |
| "NoSQL means no schema" | The schema still exists — it just lives in your application code instead of the database. |
| "NoSQL means no transactions" | Many NoSQL DBs support transactions now, often with limits. |

---

## Where this shows up in HLD

- Almost every design interview has a "which database?" moment. Interviewers want a **reason tied to the access pattern**, not a brand name.
- Good answer shape: "Orders need multi-row transactions, so Postgres. Chat messages are append-heavy and read by `(chat_id, time)`, so Cassandra. Sessions go in Redis."
- Picking NoSQL usually means you must **design around queries upfront** — say what your queries are before naming the store.
- Mentioning trade-offs (joins lost, consistency weakened, scaling gained) shows maturity.

---

## Key takeaways

- SQL = tables + joins + ACID. The safe default for related, correctness-critical data.
- NoSQL = four families (document, key-value, wide-column, graph), each optimized for one kind of access.
- Pick based on **access patterns and scale**, not hype.
- NoSQL gains easy horizontal scaling, usually by giving up joins and some consistency.
- Real systems mix several databases, each doing what it's best at.
