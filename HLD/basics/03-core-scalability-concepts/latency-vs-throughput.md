# Latency vs Throughput

## TL;DR

- **Latency** = how long **one** request takes. (Measured in ms.)
- **Throughput** = how **many** requests you finish per unit of time. (Measured in requests/sec, MB/sec.)

They are related but **not the same thing**. A system can have great throughput and terrible latency, or the other way around.

And when you measure latency, **don't look at the average** — look at percentiles (p50, p95, p99). The average hides the users who are having a bad time.

---

## The analogy that makes it click

**A highway.**

- **Latency** is how long it takes *one car* to drive from city A to city B. Depends on distance and speed limit.
- **Throughput** is how many cars *arrive* at city B per hour. Depends on how many lanes there are.

```
1 lane, 100 km/h                    6 lanes, 100 km/h
═══════════════════                 ═══════════════════
 [car] ──►                            [car] ──►
                                     [car] ──►
                                     [car] ──►   ... x6 lanes
═══════════════════                 ═══════════════════

Latency:    1 hour                  Latency:    1 hour      (same!)
Throughput: 1000 cars/hr            Throughput: 6000 cars/hr
```

Adding lanes doesn't get any single car there faster. It just lets more cars through. That's the key insight: **improving throughput doesn't automatically improve latency, and vice versa.**

Same idea with a **water pipe**: latency is how long a drop takes to travel the pipe's length; throughput is how much water comes out per second (depends on the pipe's width).

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Latency** | Time for one operation, start to finish |
| **Response time** | What the user feels: latency + queuing + processing |
| **Throughput** | Operations completed per second (RPS, QPS, TPS, MB/s) |
| **Bandwidth** | The *maximum possible* throughput of a link (the pipe's width) |
| **Percentile (pX)** | X% of requests were at or below this value |
| **Tail latency** | The slow end: p99, p99.9 |

> Bandwidth vs throughput: bandwidth is what the pipe *could* carry, throughput is what it *actually* carries.

---

## Why averages lie

Imagine 10 requests with these latencies:

```
20, 22, 21, 19, 23, 20, 22, 21, 20, 2000   (ms)
```

- **Average:** 218 ms — looks mediocre, but *no one* actually experienced 218 ms.
- **Median (p50):** ~21 ms — most users are happy.
- **Max:** 2000 ms — one user waited 2 seconds.

The average blends the happy majority with the miserable few, and describes neither. That's why engineers use **percentiles**.

---

## Percentiles: p50, p95, p99

Sort all request latencies from fastest to slowest. The **pX** is the value X% of the way along.

```
fastest ─────────────────────────────────────────────► slowest
│                    │                    │        │   │
                    p50                  p90     p95  p99
               "typical user"                    "unlucky 1 in 100"
```

| Percentile | Meaning | Example |
|---|---|---|
| **p50** (median) | Half of requests are faster than this | 40 ms |
| **p95** | 95% are faster; 1 in 20 is slower | 120 ms |
| **p99** | 99% are faster; 1 in 100 is slower | 450 ms |
| **p99.9** | 1 in 1,000 is slower | 1.8 s |

**Why p99 matters even though it's "only 1%":**

- At 10,000 requests/sec, 1% is **100 slow requests every second.**
- One page load often makes **dozens** of backend calls. If a page makes 100 calls, the chance that *at least one* hits the p99 is `1 - 0.99^100 ≈ 63%`. The p99 of a backend becomes the *typical* experience of the page.
- Your heaviest users (most data, most requests) are often the ones hitting the tail — and they're usually your most valuable.

---

## Tail latency: where the slow requests come from

The p99 is usually slow for reasons that have nothing to do with your code's normal path:

- **Garbage collection pauses** (JVM, Go, Node)
- **Queuing** — the request waited behind others
- **Cache misses** — this one had to go to disk / the database
- **Noisy neighbors** — another VM on the same host hogging CPU or disk
- **Retries and timeouts** — one slow dependency drags the whole request
- **Network hiccups** — a dropped packet means a TCP retransmit

**Common fixes:**

| Technique | Idea |
|---|---|
| **Timeouts** | Don't wait forever on a slow dependency |
| **Hedged requests** | Send the same request to 2 replicas, take whichever answers first |
| **Caching** | Remove slow paths entirely for hot data |
| **Fewer fan-out calls** | Each extra parallel call raises the odds of hitting a slow one |
| **Load shedding** | Reject some requests when overloaded, so the rest stay fast |

---

## How latency and throughput trade off

Often you can buy one by spending the other.

### Batching: more throughput, more latency

Instead of writing each row to the database immediately, collect 1,000 rows and write them in one go.

```
Without batching:  write, write, write, write ...   (each one pays full overhead)
With batching:     wait ... wait ... [WRITE 1000]   (overhead paid once)

Throughput:  ⬆ way up
Latency:     ⬆ also up — the first row waited for the batch to fill
```

Kafka producers, database bulk inserts, and network stacks (Nagle's algorithm) all do this.

### Queuing: throughput holds, latency explodes

When a server gets busier, latency doesn't grow evenly — it grows *slowly* at first, then shoots up as you approach full capacity.

```
Latency
  │                                    │
  │                                   ╱
  │                                  ╱
  │                                ╱
  │                            ╱
  │                    ╱ ─ ─
  │─ ─ ─ ─ ─ ─ ─ ─ ─
  └───────────────────────────────────┴──── Utilization
  0%                 50%        80%   100%
                                 ▲
                        "the knee" — stay below this
```

At 90%+ utilization, requests spend most of their time **waiting in line**, not being processed. Throughput is at its max, but users suffer. This is why systems are usually run at 50-70% utilization, leaving headroom.

---

## Little's Law (briefly)

A simple formula that connects the two:

```
L = λ × W

L = average number of requests in the system at once (concurrency)
λ = arrival rate (throughput, requests/sec)
W = average time each request spends in the system (latency, sec)
```

**Example:** your service handles 1,000 req/sec and each takes 200 ms.

```
L = 1000 × 0.2 = 200 requests in flight at any moment
```

So you need capacity for ~200 concurrent requests (threads, connections, workers). If latency doubles to 400 ms at the same throughput, you suddenly need 400 — which is how a slow database can exhaust your connection pool.

---

## Latency numbers every programmer should know

The classic figures (from Jeff Dean / Peter Norvig), rounded. Modern hardware is faster in places — the **orders of magnitude between rows** are what matter.

| Operation | Time | Human scale (if 1 ns = 1 sec) |
|---|---|---|
| L1 cache reference | 0.5 ns | 0.5 sec |
| Branch mispredict | 5 ns | 5 sec |
| L2 cache reference | 7 ns | 7 sec |
| Mutex lock/unlock | 25 ns | 25 sec |
| Main memory (RAM) reference | 100 ns | ~2 min |
| Compress 1 KB (fast algorithm) | 3 µs | ~1 hour |
| Send 1 KB over 1 Gbps network | 10 µs | ~3 hours |
| Random 4 KB read from SSD | 150 µs | ~2 days |
| Read 1 MB sequentially from RAM | 250 µs | ~3 days |
| Round trip within a datacenter | 500 µs | ~6 days |
| Read 1 MB sequentially from SSD | 1 ms | ~12 days |
| HDD disk seek | 10 ms | ~4 months |
| Read 1 MB sequentially from HDD | 20 ms | ~8 months |
| Packet round trip US ↔ Europe | 150 ms | ~5 years |

**What to take away from this table:**

- **Memory is ~1,000x faster than SSD, which is much faster than spinning disk.** This is why caches exist.
- **A network hop inside a datacenter (~0.5 ms) is cheap-ish; crossing an ocean (~150 ms) is not.** This is why CDNs and multi-region setups exist.
- **Sequential reads are far faster than random reads**, especially on HDD. This is why logs and LSM-trees append instead of updating in place.

---

## Latency vs throughput at a glance

| | Latency | Throughput |
|---|---|---|
| **Question it answers** | "How fast is one request?" | "How many requests can we handle?" |
| **Unit** | ms, µs | req/sec, MB/sec |
| **Who feels it** | The individual user | The system / the business |
| **Improved by** | Caching, CDNs, fewer hops, faster code | More servers, parallelism, batching |
| **Hurt by** | Distance, queuing, GC, disk I/O | Bottlenecks, locks, single-threaded parts |
| **How to report it** | Percentiles (p50/p95/p99) | Peak and sustained rate |

---

## Where this shows up in HLD

- **Requirements phase:** interviewers expect you to ask both — "What latency do we need (e.g. p99 < 200 ms)?" and "What throughput (e.g. 50k QPS peak)?" These drive every later decision.
- **Back-of-the-envelope estimates** use the latency numbers table: "a DB call is ~1-10 ms, a cache hit is sub-ms, so we cache the hot path."
- **Always state SLOs as percentiles**, not averages. Saying "p99 under 300 ms" signals experience.
- **Batching vs real-time** is a classic trade-off: analytics pipelines batch for throughput; payment and chat systems prioritize latency.
- **Little's Law** helps size connection pools, thread pools, and worker counts.
- **Fan-out services** (news feed, search) are where tail latency hurts most — mention timeouts and hedged requests.

---

## Key takeaways

- **Latency = time per request. Throughput = requests per second.** Improving one doesn't automatically improve the other.
- **Averages lie.** Measure and set goals using p50, p95, and p99.
- Tail latency gets amplified when one request fans out to many backend calls.
- Batching raises throughput at the cost of latency; high utilization makes latency explode from queuing.
- Little's Law (`L = λ × W`) links concurrency, throughput, and latency.
- Memory ≪ SSD ≪ disk ≪ cross-ocean network — design around those orders of magnitude.
