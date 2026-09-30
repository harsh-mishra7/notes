# Single Point of Failure (SPOF)

## Brief

A **single point of failure** is any one component that, if it breaks, **takes the whole system down with it.**

The fix is **redundancy**: have more than one of everything important, plus a way to **detect** failures (health checks) and **switch over** automatically (failover).

The skill in system design is looking at a diagram and asking of every box: *"What happens if this one dies?"* If the answer is "everything stops," you've found a SPOF.

---

## The analogy that makes it click

**Old-style Christmas lights wired in series.**

One bulb burns out, and the entire string goes dark. You then spend an hour testing every bulb to find which one it was. That one bulb was a single point of failure.

Modern lights are wired in **parallel** — one bulb dies, the rest keep shining. Same idea in systems: you want failures to be *local* ("one bulb is out") instead of *global* ("the whole house is dark").

Other everyday SPOFs:

- The one person on your team who knows how the deploy works.
- A building with only one entrance during a fire.
- A single road into a town — close it, and the town is cut off.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **SPOF** | A component whose failure stops the whole system |
| **Redundancy** | Having extra copies of a component |
| **Failover** | Automatically switching to a backup when the main one fails |
| **Health check** | A regular "are you alive?" probe |
| **Replication** | Keeping copies of data on multiple machines |
| **Availability Zone (AZ)** | A separate datacenter (own power, cooling, network) within a cloud region |
| **Region** | A geographic area (e.g. `us-east-1`) containing multiple AZs |

---

## How to spot SPOFs in a diagram

Walk through the diagram and, for every box and every arrow, ask: **"If this disappears, does the system still work?"**

```
User ──► [ DNS ] ──► [ Load Balancer ] ──► [ App Server ] ──► [ Database ]
            ?               ?                    ?                  ?
```

Things to look for:

- **Any box drawn once** on the request path.
- **Anything that holds state** — databases, caches used as the source of truth, file storage.
- **Shared infrastructure** that's easy to overlook: one rack, one power supply, one network switch, one AZ, one region.
- **External dependencies** — a single payment provider, a single DNS provider, a third-party auth service.
- **Humans and processes** — one person with production access, a manual failover step, an expired TLS certificate no one owns.

> A "redundant" pair isn't redundant if both copies share the same hidden dependency. Two servers on the same power strip are one SPOF.

---

## Redundancy: active-active vs active-passive

Having a second copy is step one. How the copies work together is step two.

### Active-passive (primary / standby)

One node does all the work. The other sits idle (or just keeps a copy of data), ready to take over.

```
           ┌──────────────┐
traffic ──►│  PRIMARY  ✅ │  handles everything
           └──────┬───────┘
                  │ replicates data / heartbeat
           ┌──────▼───────┐
           │  STANDBY  -- │  waits; takes over if primary dies
           └──────────────┘
```

### Active-active

All nodes handle traffic at the same time. If one dies, the others absorb its share.

```
                 ┌──────────┐
            ┌──► │ Node A ✅│
traffic ──► LB   ├──────────┤
            └──► │ Node B ✅│   both serving live traffic
                 └──────────┘
```

| | Active-passive | Active-active |
|---|---|---|
| **Resource use** | Standby is idle (wasted capacity) ❌ | All capacity used ✅ |
| **Failover time** | Seconds to minutes (standby must be promoted) | Near-instant (others already serving) ✅ |
| **Complexity** | Simpler ✅ | Harder — especially for writes to data ❌ |
| **Data consistency** | Easy, one writer ✅ | Hard if multiple nodes accept writes (conflicts) ❌ |
| **Typical use** | Relational DB primary + standby | Stateless app servers, CDN edges, multi-region reads |

**Capacity warning for active-active:** if two nodes each run at 70% load and one dies, the survivor needs 140%. It will fall over too. Size so that the remaining nodes can carry the full load (**N+1** redundancy).

---

## Failover

Failover is the act of switching traffic from a failed component to a healthy one.

```
1. Health check fails 3 times in a row  → primary marked DOWN
2. Standby is promoted to primary        → starts accepting writes
3. Traffic is re-pointed                  → via LB, DNS, or a floating/virtual IP
4. Old primary is fenced off              → so it can't come back and also accept writes
```

| Type | How | Trade-off |
|---|---|---|
| **Automatic** | System detects and switches on its own | Fast, but risk of false alarms |
| **Manual** | A human decides and triggers it | Safe, but slow (high MTTR) |

**The split-brain problem:** if the network between primary and standby breaks, the standby might think the primary is dead and promote itself — while the primary is actually still alive. Now two nodes both accept writes and data diverges. Protection includes **fencing** (forcibly shutting the old primary out) and **quorum** (a majority of nodes must agree before anyone is promoted).

---

## Health checks

You can't fail over from something you don't know is broken.

| Type | What it checks | Example |
|---|---|---|
| **Liveness** | Is the process running at all? | TCP connect on port 8080 |
| **Readiness** | Can it actually serve requests right now? | `GET /health` returns 200 only if DB connection is OK |
| **Heartbeat** | Nodes ping each other regularly | Standby expects a ping from primary every 1s |

Tips:

- **Require several consecutive failures** before marking a node down, so one slow response doesn't cause a failover.
- **Keep health endpoints cheap.** A health check that runs a heavy query can itself overload the system.
- **Be careful checking dependencies.** If every app server's health check fails whenever the DB is slow, the LB removes *all* app servers at once — and turns a slow DB into a full outage.

---

## Replication (for stateful components)

Stateless servers are easy to duplicate: start another one. **Data is the hard part** — the backup copy is useless if it doesn't have the data.

| Style | How | Trade-off |
|---|---|---|
| **Synchronous** | Primary waits for replica to confirm before acknowledging a write | No data loss on failover, but slower writes |
| **Asynchronous** | Primary acknowledges immediately, replica catches up later | Fast, but a failover can lose the last few writes |

Most databases (PostgreSQL, MySQL) default to asynchronous replication and offer synchronous as an option.

---

## Multi-AZ and multi-region

Redundancy only helps if the copies don't fail *together*. So you spread them out.

```
                         REGION (us-east-1)
   ┌──────────────────────────────────────────────────────┐
   │   ┌──── AZ-a ────┐   ┌──── AZ-b ────┐   ┌── AZ-c ──┐  │
   │   │ App, App     │   │ App, App     │   │ App, App │  │
   │   │ DB primary   │──►│ DB standby   │   │          │  │
   │   └──────────────┘   └──────────────┘   └──────────┘  │
   └──────────────────────────────────────────────────────┘
                              │ async replication
                              ▼
                     REGION (eu-west-1)  ← survives a whole-region outage
```

| Level | Protects against | Cost / complexity |
|---|---|---|
| **Multiple servers** | One machine dying | Low |
| **Multiple AZs** | A datacenter losing power / network | Moderate — the standard for production |
| **Multiple regions** | An entire region going down, natural disasters | High — cross-region latency, data consistency, cost |

---

## Removing SPOFs step by step

Start with the simplest possible app and fix one SPOF at a time.

**Step 0 — Everything on one box.**

```
User ──► [ Server: app + DB ]
```
SPOFs: literally everything.

**Step 1 — Split app and DB, add more app servers behind a load balancer.**

```
              ┌──► [ App 1 ] ──┐
User ──► LB ──┤                ├──► [ DB ]
              └──► [ App 2 ] ──┘
```
✅ An app server can die. ❌ The LB and DB are still SPOFs.

**Step 2 — Make the load balancer redundant.**

```
          ┌─ LB (active) ─┐        (floating IP, or a managed LB
User ──►  │               │         like AWS ALB that's redundant
          └─ LB (standby)─┘         under the hood)
```
✅ LB can die. ❌ DB is still a SPOF.

**Step 3 — Add a database standby with replication and automatic failover.**

```
              ┌──► [ App 1 ] ──┐      ┌──► [ DB primary ]
User ──► LBs ─┤                ├──────┤         │ replication
              └──► [ App 2 ] ──┘      └──► [ DB standby ]
```
✅ DB primary can die. ❌ All of this is in one datacenter.

**Step 4 — Spread across availability zones.**

App servers in AZ-a and AZ-b, DB primary in one AZ and standby in the other. ✅ A whole datacenter can go dark.

**Step 5 (only if needed) — Multi-region, plus redundant DNS.**

✅ A whole region can fail. This is expensive and complex; most companies stop at Step 4.

| Step | Can survive the loss of… |
|---|---|
| 0 | Nothing |
| 1 | One app server |
| 2 | + a load balancer |
| 3 | + the database primary |
| 4 | + an entire datacenter (AZ) |
| 5 | + an entire region |
