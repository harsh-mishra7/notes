# Availability vs Reliability

## Brief

- **Availability** = is the system **up and responding** right now? (Measured as % of time it's usable.)
- **Reliability** = does the system **do the right thing, consistently, without failing**? (Measured as how long it runs between failures.)
- **Durability** = once data is saved, does it **stay saved**? (Measured as the chance of never losing it.)

A system can be available but unreliable (it responds, but sometimes with wrong answers or errors). Highly reliable systems tend to be highly available, but not always the other way around.

Availability is usually expressed in **"nines"**: 99.9% ("three nines"), 99.99% ("four nines"), and so on.

---

## The analogy that makes it click

**Think of two cars.**

- **Car A** starts every morning without fail, but breaks down on the highway once a week. You can always *get in and go* (high availability), but you can't *trust it to get you there* (low reliability).
- **Car B** never breaks down mid-trip, but is in the repair shop 2 days a month for scheduled maintenance. Very reliable when running, but not always available.

What you want is a car that's both: always ready, and never fails mid-trip. And if you leave something in the trunk, it should still be there next year — that's **durability**.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Availability** | % of time the system is working and reachable |
| **Reliability** | Probability the system works correctly over a period without failing |
| **Durability** | Probability that stored data is never lost |
| **Downtime** | Time the system is not available |
| **MTBF** | Mean Time Between Failures — how long it usually runs before breaking |
| **MTTR** | Mean Time To Repair/Recover — how long it takes to fix once broken |

---

## Availability vs reliability: the difference

| | Availability | Reliability |
|---|---|---|
| **Question** | "Can I use it right now?" | "Will it keep working correctly?" |
| **Measured as** | Uptime % | MTBF, failure rate, error rate |
| **Improved by** | Redundancy, fast failover, low MTTR | Better code, testing, fewer bugs, higher MTBF |
| **Example failure** | Site returns "503 Service Unavailable" | Site is up but charges customers twice |

A key subtlety: **you can make an unreliable system highly available** by having lots of redundant copies that fail over quickly. Each component crashes often, but the system as a whole rarely goes down. This is how most large-scale systems are built — assume parts fail, and design around it.

---

## The nines table

Availability is written as a percentage of uptime. Each extra "9" cuts allowed downtime by 10x.

| Availability | Nickname | Downtime / year | Downtime / month | Downtime / day |
|---|---|---|---|---|
| 99% | two nines | 3.65 days | 7.31 hours | 14.4 minutes |
| 99.9% | three nines | 8.77 hours | 43.8 minutes | 1.44 minutes |
| 99.99% | four nines | 52.6 minutes | 4.38 minutes | 8.64 seconds |
| 99.999% | five nines | 5.26 minutes | 26.3 seconds | 0.86 seconds |

(Month = 1/12 of a year, ~30.4 days.)

**How to read this:**

- **99.9%** is a common target for normal web apps. 43 minutes a month is enough for an occasional bad deploy.
- **99.99%** means a human can barely react before the budget is gone — you need **automated** failover.
- **99.999%** is telecom / critical-infrastructure territory. Very expensive; every change is risky.

Each extra nine costs roughly an order of magnitude more effort and money. Pick the one your business actually needs.

---

## SLA, SLO, SLI

Three terms people mix up constantly. They build on each other.

| Term | Stands for | What it is | Example |
|---|---|---|---|
| **SLI** | Service Level **Indicator** | The actual **measurement** | "99.95% of requests succeeded this month" |
| **SLO** | Service Level **Objective** | The internal **target** for that measurement | "99.9% of requests should succeed" |
| **SLA** | Service Level **Agreement** | A **contract** with customers, with penalties | "If we drop below 99.5%, you get a 10% refund" |

```
SLI  =  what you measure     (the speedometer reading)
SLO  =  what you aim for     (your own speed rule)
SLA  =  what you promise     (the speed limit — break it and you pay a fine)
```

**Rule:** SLA is always looser than SLO. You set an internal goal stricter than your public promise, so you get warned before you owe anyone money.

**Error budget:** if your SLO is 99.9%, you're *allowed* 0.1% failure — about 43 minutes a month. That's your budget. Spend it on risky deploys and experiments; when it's used up, freeze releases and focus on stability. (This idea comes from Google's SRE practice.)

---

## Availability math: series vs parallel

Real systems are made of many components. How they're connected changes the total availability dramatically.

### In series (all must work)

A request goes through a load balancer, then an app server, then a database. If **any** one fails, the request fails.

```
User ──► [ LB 99.9% ] ──► [ App 99.9% ] ──► [ DB 99.9% ]

Total = A1 × A2 × A3
      = 0.999 × 0.999 × 0.999
      ≈ 0.997   →  99.7%
```

**Chaining components always makes things worse.** Three "three nines" parts give you less than three nines overall. Every extra dependency in the critical path lowers availability.

### In parallel (only one needs to work)

Two redundant app servers; the system works as long as **at least one** is up.

```
              ┌──► [ App A 99% ] ──┐
User ──► LB ──┤                    ├──► ...
              └──► [ App B 99% ] ──┘

Chance both fail = (1 - 0.99) × (1 - 0.99) = 0.01 × 0.01 = 0.0001
Total = 1 - 0.0001 = 0.9999  →  99.99%
```

Formula:

```
Total = 1 - (1 - A1) × (1 - A2) × ...
```

**Redundancy is how you buy nines.** Two 99% servers in parallel give 99.99%. (This assumes they fail *independently* — if both sit on the same rack and the rack loses power, the math doesn't hold.)

### Putting it together

```
User ──► LB ──► [App A | App B] ──► [DB primary | DB replica]
        99.99%    99.99%                  99.99%

Total ≈ 0.9999 × 0.9999 × 0.9999 ≈ 99.97%
```

Series makes things worse; parallel makes things better. Good design keeps the series chain short and makes every link in it redundant.

---

## MTBF and MTTR

Two numbers that together determine availability:

```
                MTBF
Availability = ─────────────
               MTBF + MTTR
```

**Example:** a server fails on average every 1,000 hours (MTBF), and takes 1 hour to fix (MTTR).

```
Availability = 1000 / (1000 + 1) ≈ 99.9%
```

There are two ways to improve this:

| Lever | How | Example |
|---|---|---|
| **Increase MTBF** (fail less often) | Better hardware, testing, code review | Fewer bugs reach production |
| **Decrease MTTR** (recover faster) | Monitoring, alerts, auto-failover, quick rollbacks | Failover in 30 seconds instead of 30 minutes |

In practice, **reducing MTTR is usually easier and cheaper** than preventing all failures. Cut MTTR from 1 hour to 6 minutes and the same server goes from ~99.9% to ~99.99%.

---

## Durability: the one people forget

Availability is about **time**. Durability is about **data**.

- A database can be *unavailable* for an hour (down) but *durable* (no data lost when it comes back).
- A cache can be *highly available* but *not durable* (restart it and everything is gone — by design).

Durability is usually quoted with even more nines. AWS S3, for example, is designed for **99.999999999% (11 nines)** durability — if you store 10 million objects, you'd expect to lose one about every 10,000 years.

How durability is achieved:

| Technique | Idea |
|---|---|
| **Replication** | Keep multiple copies on different machines / zones |
| **Write-ahead log (WAL)** | Write to a log on disk before confirming, so a crash can replay it |
| **Erasure coding** | Split data into chunks + parity, so lost chunks can be rebuilt |
| **Backups** | Periodic snapshots, stored somewhere else |

> Replication is not a backup. If someone runs `DELETE FROM users`, replication faithfully deletes it everywhere. You still need point-in-time backups.
