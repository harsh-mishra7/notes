# Caching

## Brief

**A cache is a small, fast storage layer that keeps copies of data you'll probably need again.**

Reading from a database on disk might take 5-50 ms. Reading the same thing from memory (RAM) takes well under 1 ms. If the same data is read over and over, keep a copy somewhere fast and skip the slow trip.

The catch: a cache is a **copy**, and copies can go **stale**. Almost everything hard about caching is about keeping the copy correct enough.

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

---

## The analogy that makes it click

**A cache is the notepad on your desk.**

The full answer lives in a filing cabinet in the basement (the database). Walking down there every time someone asks for a customer's phone number is slow. So you jot the ones you're asked about most often on a notepad at your desk.

- Someone asks for a number that's **on the notepad** → instant answer. → **cache HIT**
- Someone asks for one that **isn't** → walk to the basement, get it, and write it on the notepad for next time. → **cache MISS**
- The notepad is small, so when it's full you **cross out** the ones nobody's asked about in a while. → **eviction**
- If a customer changes their number, your notepad is now **wrong** until you fix it. → **invalidation**

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Cache hit** | Data found in the cache. Fast. |
| **Cache miss** | Data not in the cache — go to the source (DB), usually store it after. |
| **Hit ratio** | hits / (hits + misses). The main health metric of a cache. |
| **TTL** | Time To Live — how long an entry stays before it expires. |
| **Eviction** | Removing entries when the cache is full. |
| **Invalidation** | Removing/updating an entry because the source data changed. |
| **Stale data** | A cached copy that no longer matches the source. |

---

## Why cache?

| Benefit | Why it matters |
|---|---|
| **Lower latency** | RAM is ~100x+ faster than a disk-backed DB query, and much faster than a network call to another service |
| **Less load on the database** | If 90% of reads hit the cache, the DB handles 10x fewer reads |
| **Handle traffic spikes** | Cache absorbs bursts the DB couldn't survive |
| **Save money** | Fewer DB replicas, fewer calls to paid third-party APIs |
| **Avoid recomputation** | Store the result of an expensive calculation (feed ranking, reports) |

Caching works because real traffic is **skewed**: a small fraction of data (popular posts, top products, celebrity profiles) gets most of the reads.

---

## Where caches live

Caching isn't one box — it happens at every layer between the user and the data.

```
 [ Browser cache ] ──► [ CDN ] ──► [ Load balancer / reverse proxy cache ]
                                              │
                                              ▼
                              [ App server  +  in-memory cache ]
                                              │
                                              ▼
                              [ Distributed cache: Redis / Memcached ]
                                              │
                                              ▼
                              [ Database  +  its own buffer cache ]
```

| Layer | What it caches | Example |
|---|---|---|
| **Browser** | Static files, API responses per `Cache-Control` | Your CSS file not re-downloaded on every page |
| **CDN** | Static assets near users | Images served from a Tokyo edge server |
| **Reverse proxy** | Whole HTTP responses | Nginx caching a product page for 30 s |
| **App / in-memory** | Objects inside the app process | A Python dict or Guava/Caffeine cache of config |
| **Distributed cache** | Shared key-value data for all app servers | Redis, Memcached |
| **Database cache** | Recently-read disk pages kept in RAM | Postgres shared buffers, MySQL InnoDB buffer pool |

### In-memory (local) vs distributed cache

```
LOCAL (in each app server)            DISTRIBUTED (shared)

 App 1 [cache]   App 2 [cache]         App 1    App 2    App 3
 App 3 [cache]                            \       |       /
                                           [   Redis   ]
 Each has its own copy                 One shared copy
```

| | **Local in-memory** | **Distributed (Redis/Memcached)** |
|---|---|---|
| Speed | Fastest (no network) | Very fast (~sub-ms network hop) |
| Shared across servers | ❌ No — each server has its own | ✅ Yes |
| Consistency | Servers can disagree | One source of cached truth |
| Survives app restart | ❌ No | ✅ Yes |
| Size | Limited by one server's RAM | Can scale across a cluster |

Many systems use **both**: a tiny local cache for super-hot data, Redis for everything else.

---

## Caching strategies (how reads and writes flow)

### 1. Cache-aside (lazy loading) — the most common

The **app** manages the cache. The cache doesn't talk to the DB itself.

```
READ:
  App ──► Cache: have "user:42"?
            ├─ HIT  → return it
            └─ MISS → App ──► DB: read user 42
                      App ──► Cache: store "user:42" (with TTL)
                      return it

WRITE:
  App ──► DB: update user 42
  App ──► Cache: DELETE "user:42"   (next read reloads fresh data)
```

✅ Only caches what's actually requested. Cache down? App still works (just slower).
❌ First read of every key is a miss. App code has to handle the logic.

### 2. Read-through

Like cache-aside, but the **cache itself** loads from the DB on a miss. The app only ever talks to the cache.

```
App ──► Cache ──(on miss)──► DB
```

✅ Simpler app code. ❌ Needs a cache library/provider that supports it.

### 3. Write-through

Every write goes to the **cache and the DB together**, synchronously.

```
App ──► Cache ──► DB     (write finishes only when both are updated)
```

✅ Cache is always fresh. ❌ Writes are slower; you may cache data nobody reads.

### 4. Write-back (write-behind)

Write to the **cache only**, return immediately. The cache flushes to the DB later, in batches.

```
App ──► Cache  ✅ done (fast)
          └── ... later, async ...──► DB
```

✅ Very fast writes, great for write-heavy workloads (counters, metrics).
❌ **Risk of data loss** if the cache dies before flushing.

### 5. Write-around

Writes go **straight to the DB**, skipping the cache. The cache only fills on reads.

```
WRITE: App ──► DB             (cache untouched)
READ:  cache-aside as usual
```

✅ Doesn't flood the cache with data that's written once and rarely read (logs, uploads).
❌ Recently written data is a miss on first read.

### Strategy summary

| Strategy | Write goes to | Read miss handled by | Best for | Main risk |
|---|---|---|---|---|
| Cache-aside | DB (then delete cache key) | App | General read-heavy apps (default choice) | Stale data if invalidation is missed |
| Read-through | — | Cache | Same as above, cleaner code | Provider lock-in |
| Write-through | Cache + DB together | — | Data that's read right after being written | Slower writes |
| Write-back | Cache, DB later | — | Write-heavy, loss-tolerant data | Data loss on cache crash |
| Write-around | DB only | App/cache | Write-once, rarely-read data | Misses on fresh data |

---

## Eviction policies (what to remove when full)

| Policy | Removes | Good for |
|---|---|---|
| **LRU** — Least Recently Used | The item not accessed for the longest time | General purpose. The default in most caches. |
| **LFU** — Least Frequently Used | The item accessed the fewest times | Stable popularity (some items are always hot) |
| **FIFO** — First In, First Out | The oldest inserted item, regardless of use | Simple; rarely the best choice |
| **TTL** — Time To Live | Anything past its expiry time | Data that goes stale on a schedule |

```
LRU example (capacity 3):

get A     [A]
get B     [A, B]
get C     [A, B, C]
get A     [B, C, A]      ← A moved to "most recent"
get D     [C, A, D]      ← full: B was least recently used → evicted
```

TTL is usually **combined** with LRU/LFU: entries expire after their TTL, *and* LRU kicks in if memory fills up first. Redis supports all of these via its `maxmemory-policy` setting.

---

## Cache invalidation

When the source data changes, the cached copy is wrong. Ways to deal with it:

| Approach | How | Trade-off |
|---|---|---|
| **TTL expiry** | Let entries expire after N seconds | Simple; data can be stale for up to N seconds |
| **Delete on write** | On update, delete the cache key | Very common with cache-aside; next read reloads |
| **Update on write** | On update, write the new value into the cache | Risk of race conditions writing an old value |
| **Event-based** | DB change events (CDC) or a message trigger invalidation | Accurate, more moving parts |

**Why "delete" is usually preferred over "update":** if two writes race, updating the cache can leave the *older* value sitting there. Deleting just forces the next read to fetch the truth.

**Always set a TTL anyway** — it's your safety net if an invalidation is ever missed.

---

## Cache stampede (thundering herd)

A very popular key expires. In the next millisecond, 10,000 requests all miss **at the same time**, and all 10,000 hit the database to rebuild the same value.

```
 "homepage" key expires at 12:00:00
                │
   10,000 requests ─► MISS ─► all go to DB at once ─► DB overloaded ❌
```

Fixes:

| Fix | How it works |
|---|---|
| **Locking / request coalescing** | Only the first request rebuilds the value; others wait for it or get the old value |
| **Stale-while-revalidate** | Serve the slightly old value while one background job refreshes it |
| **Early/probabilistic refresh** | Refresh a hot key a little *before* it expires |
| **TTL jitter** | Add randomness to TTLs (e.g. 300 s ± 30 s) so many keys don't expire at the same moment |

Related problems: **cache penetration** (requests for keys that don't exist at all, so they always miss — fix by caching "not found" briefly or using a Bloom filter) and **cache avalanche** (the whole cache goes down or many keys expire together).

---

## Hit ratio

```
hit ratio = hits / (hits + misses)
```

| Hit ratio | Meaning |
|---|---|
| 95%+ | Great — DB sees only 1 in 20 reads |
| 80% | Decent |
| < 50% | The cache may not be earning its keep — wrong keys, TTL too short, or cache too small |

A small change matters a lot: going from 90% to 99% hit ratio cuts DB reads **10x** (from 10% to 1% of traffic).
