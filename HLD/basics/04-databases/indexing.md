# Database Indexing

## TL;DR

An **index** is a separate, sorted data structure that helps the database **find rows without scanning the whole table**.

Without an index, `WHERE email = 'asha@x.com'` means checking every single row. With an index on `email`, the database jumps almost straight to it.

The catch: every index **costs extra storage** and makes **every write slower**, because the index must be updated too. Indexes are a trade: faster reads, slower writes.

---

## The analogy that makes it click

**An index is the index at the back of a textbook.**

You want to find "photosynthesis" in a 900-page book.

- **No index** → read every page until you find it. (**Full table scan**)
- **With index** → flip to the back, find "photosynthesis → pages 212, 415", jump there. (**Index lookup**)

The book's index is sorted alphabetically, takes up extra pages, and has to be rewritten every time the author adds a chapter. Same trade-offs as a database index.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Full table scan** | Reading every row to find matches. O(n). |
| **Index** | A sorted structure mapping column values → row locations. |
| **Primary / clustered index** | The table's rows are physically stored in this order (e.g. by primary key in MySQL InnoDB). |
| **Secondary index** | An extra index that points back to the row (or its primary key). |
| **Selectivity** | How well a column narrows down results. `email` = high, `gender` = low. |
| **Query plan** | The database's chosen strategy. See it with `EXPLAIN`. |

---

## What difference does it make?

```
Table: users (10,000,000 rows)
Query: SELECT * FROM users WHERE email = 'asha@x.com';

NO INDEX                           WITH B-TREE INDEX ON email
check row 1        ✗               root   → "a..m" branch
check row 2        ✗               branch → "as..at" leaf
...                                leaf   → asha@x.com → row #8,431,002
check row 10,000,000               fetch row
~10M comparisons                   ~3-4 page reads  ✅
```

That's the difference between seconds and a millisecond.

---

## B-tree index (the default)

Almost every relational database (Postgres, MySQL, SQL Server) uses a **B-tree** (technically usually a B+ tree) by default.

It's a **balanced, sorted tree** that is very wide and very shallow. Each node holds hundreds of keys, so even a billion rows need only ~4 levels.

```
                      [ M ]
                   /         \
          [ D  H ]             [ R  W ]
         /   |   \            /   |   \
     [A-C] [E-G] [I-L]    [N-Q] [S-V] [X-Z]   ← leaf nodes, linked left-to-right
       ↓     ↓     ↓        ↓     ↓     ↓
     row pointers ...
```

Because it's **sorted**, a B-tree supports:

- Exact match: `WHERE id = 42`
- Range: `WHERE age BETWEEN 20 AND 30`
- Prefix: `WHERE name LIKE 'As%'` (but **not** `LIKE '%sha'`)
- Sorting: `ORDER BY created_at` without an extra sort step
- Min / max quickly

Lookups, inserts and deletes are all **O(log n)**.

---

## Hash index

A **hash index** runs the key through a hash function and stores it in a bucket — just like a hash map.

```
hash("asha@x.com") = 7  →  bucket 7  →  row #8,431,002
```

| | B-tree | Hash |
|---|---|---|
| Exact match (`=`) | ✅ O(log n) | ✅ O(1) average |
| Range (`<`, `>`, `BETWEEN`) | ✅ | ❌ hash destroys order |
| `ORDER BY` | ✅ | ❌ |
| Prefix `LIKE 'abc%'` | ✅ | ❌ |
| Typical use | Default for almost everything | In-memory stores, key-value lookups |

In practice, B-trees are close enough to O(1) for exact matches that most tables just use B-trees. Hash indexes show up more in key-value stores (Redis, in-memory hash tables) than in relational tables.

---

## LSM trees (briefly)

B-trees update data **in place**, which means random disk writes. For write-heavy systems, that becomes the bottleneck.

**LSM tree** (Log-Structured Merge tree) flips the approach: **never update in place, always append**.

```
Write ──► MemTable (sorted, in memory)
              │  when full, flush
              ▼
          SSTable 1 (sorted, immutable file on disk)
          SSTable 2
          SSTable 3
              │  background "compaction" merges them
              ▼
          Bigger merged SSTable
```

- **Writes are very fast** — just add to memory + a sequential log.
- **Reads can be slower** — a key might be in several files; bloom filters help skip files that definitely don't have it.
- Used by: **Cassandra, RocksDB, LevelDB, HBase, ScyllaDB**.

| | B-tree | LSM tree |
|---|---|---|
| Writes | Slower (random I/O) | ✅ Faster (sequential) |
| Reads | ✅ Faster, predictable | Slower, may check multiple files |
| Best for | Read-heavy, general purpose | Write-heavy (logs, metrics, events) |

---

## Composite indexes (and why column order matters)

A **composite index** covers multiple columns, e.g. `INDEX(country, city, age)`.

It's sorted like a phone book: first by `country`, then by `city` within each country, then by `age` within each city.

```
(IN, Delhi,  22)
(IN, Delhi,  35)
(IN, Pune,   19)
(IN, Pune,   41)
(US, Boston, 28)
...
```

This leads to the **leftmost prefix rule**: the index can be used only if your query filters on the columns **from the left**.

| Query `WHERE ...` | Uses index `(country, city, age)`? |
|---|---|
| `country = 'IN'` | ✅ |
| `country = 'IN' AND city = 'Pune'` | ✅ |
| `country = 'IN' AND city = 'Pune' AND age = 19` | ✅ fully |
| `city = 'Pune'` | ❌ skips `country` |
| `age = 19` | ❌ |
| `country = 'IN' AND age = 19` | ⚠️ partially (only `country` part) |

Same as a phone book sorted by last name, then first name: easy to find all "Sharma"s, useless for finding everyone named "Ravi".

**How to order columns:**

1. Columns used with **equality** (`=`) go first.
2. Columns used for **range** or **sorting** go last (after a range, later columns can't be used for seeking).
3. Among equality columns, put the ones your queries **always** include first.

---

## Covering indexes

Normally an index lookup is two steps: find the entry in the index, then go fetch the full row from the table.

If the index **already contains every column the query needs**, the database can skip the second step. That's a **covering index**.

```sql
-- Index: (user_id, created_at, total)
SELECT created_at, total
FROM orders
WHERE user_id = 42;
-- Everything needed is in the index → no table lookup ("index-only scan")
```

```
Normal:    index ──► row pointer ──► table row ──► result
Covering:  index ─────────────────────────────────► result   ✅ fewer reads
```

Great for hot queries. But the index gets bigger, so don't cover every query.

---

## The cost of indexes

Indexes are not free. Every index is a copy of some of your data that must stay in sync.

| Cost | Why |
|---|---|
| **Slower writes** | Every `INSERT`, `UPDATE`, `DELETE` must also update every affected index. 5 indexes ≈ 6 writes. |
| **More storage** | Indexes can be as large as the table itself, sometimes larger. |
| **More memory pressure** | Indexes work best when they fit in RAM. Too many compete for cache. |
| **Maintenance** | Indexes can bloat/fragment and need rebuilding over time. |

---

## When NOT to index

- **Low-selectivity columns** — `is_active` (true/false), `gender`. The index narrows down to half the table; a scan is often just as fast.
- **Tiny tables** — a few hundred rows fit in one or two pages. Scanning is instant.
- **Write-heavy tables rarely queried** — e.g. raw event logs you only batch-process.
- **Columns you never filter, join, or sort on.**
- **Queries that wrap the column in a function** — `WHERE LOWER(email) = ...` won't use a plain index on `email` (use an expression index instead).
- **Duplicate indexes** — `(a)` is redundant if you already have `(a, b)`.

**Rule of thumb:** index what your **real queries** filter, join, and sort on. Check with `EXPLAIN`. Remove indexes nobody uses.

---

## Where this shows up in HLD

- When you define a table in a design, say which **indexes** support your main queries: "orders indexed on `(user_id, created_at)` for the order history page".
- **Read-heavy vs write-heavy** decides the storage engine: B-tree (Postgres/MySQL) vs LSM (Cassandra/RocksDB).
- In NoSQL, the partition/sort key *is* the index. In DynamoDB, extra access patterns need **secondary indexes** (GSIs), which cost extra writes too.
- Search features ("find products containing 'red shoes'") need a different kind of index — an **inverted index** (Elasticsearch), not a B-tree.

---

## Key takeaways

- An index is a sorted side-structure that turns full scans into fast lookups.
- B-tree is the default: supports equality, ranges, and sorting. Hash supports only equality.
- LSM trees trade read speed for very fast writes; used by Cassandra and RocksDB.
- Composite index column order matters: leftmost prefix rule, equality first, range last.
- Every index slows writes and costs storage — index for real queries only.
