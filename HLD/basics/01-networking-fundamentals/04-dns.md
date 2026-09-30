# DNS

## Brief

**DNS = Domain Name System.** The internet's phone book.

Humans remember `google.com`. Computers need an IP address like `142.250.183.14`. DNS translates one into the other.

It's not one big server — it's a **hierarchy** of servers (root → `.com` → `google.com`'s own servers), with **caching at every layer** so that most lookups never travel the whole chain.

And because DNS decides *which IP* you get, it's also a cheap tool for **load balancing** and **sending users to the nearest data center**.

---

## The analogy that makes it click

**DNS is asking for directions in a huge library.**

You want the book "shop.com". You ask the **front desk** (your resolver). They don't know every book, but they know how to find out:

1. They ask the **head librarian** (root): "Where are the `.com` books?" → "Floor 3."
2. They go to **floor 3** (TLD server): "Where's `shop.com`?" → "Shelf 12, ask that shelf's keeper."
3. They ask **shelf 12's keeper** (authoritative server): "Where exactly is `shop.com`?" → "Aisle 93.184.216.34."

Then the front desk **writes it on a sticky note** (cache), so the next person asking gets the answer instantly.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Domain** | A human-readable name: `api.shop.com` |
| **Recursive resolver** | The server that does the lookup on your behalf (your ISP's, or `8.8.8.8`, `1.1.1.1`) |
| **Root server** | Top of the tree. Knows where each TLD's servers are. |
| **TLD server** | Top-Level Domain server: handles `.com`, `.org`, `.in`, etc. |
| **Authoritative server** | Holds the real records for a domain. The final answer. |
| **Record** | One entry in DNS, like "`shop.com` → `93.184.216.34`" |
| **TTL** | Time To Live — how long a cached answer may be reused |

---

## How a name gets resolved

```
             ┌─────────────────────────────┐
             │         Your device         │
             │    (browser + OS cache)     │
             └────────┬───────────┬────────┘
          1 shop.com? │           ▲ 8 93.184.216.34
                      ▼           │
┌─────────────────────┴───────────┴─────────────────────┐
│                  Recursive resolver                   │
│               (ISP / 8.8.8.8 / 1.1.1.1)               │
└───────┬───────────────────┬───────────────────┬───────┘
        │                   │                   │
        │ 2 ↓ shop.com?     │ 4 ↓ shop.com?     │ 6 ↓ shop.com?
        │ 3 ↑ go to .com    │ 5 ↑ ns1.shop.com  │ 7 ↑ 93.184.216.34
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│     Root      │   │   .com TLD    │   │ Authoritative │
│    servers    │   │    servers    │   │ ns1.shop.com  │
└───────────────┘   └───────────────┘   └───────────────┘
```

Step by step:

1. Browser asks the OS, OS asks the **recursive resolver**.
2. Resolver asks a **root server**: "Who handles `.com`?"
3. Root replies with the `.com` **TLD servers**.
4. Resolver asks the TLD server: "Who handles `shop.com`?"
5. TLD replies with `shop.com`'s **authoritative nameservers** (its NS records).
6. Resolver asks the authoritative server: "What's the IP of `shop.com`?"
7. Authoritative server answers: `93.184.216.34`.
8. Resolver caches it and returns it to you.

Two kinds of queries are happening here:

| | Who does it | Behavior |
|---|---|---|
| **Recursive** | You → resolver | "Get me the final answer, whatever it takes." |
| **Iterative** | Resolver → root/TLD/authoritative | "Tell me what you know, or who to ask next." |

DNS mostly uses **UDP port 53** — one small question, one small answer, no handshake. It falls back to TCP for large responses.

---

## Record types

| Type | Maps | Example | Used for |
|---|---|---|---|
| **A** | Name → IPv4 | `shop.com → 93.184.216.34` | The basic lookup |
| **AAAA** | Name → IPv6 | `shop.com → 2606:2800:220:1::248` | Same, for IPv6 |
| **CNAME** | Name → another name | `www.shop.com → shop.com` | Aliases; pointing to a CDN or SaaS host |
| **MX** | Domain → mail server (+ priority) | `shop.com → 10 mail.shop.com` | Where to deliver email |
| **NS** | Domain → its nameservers | `shop.com → ns1.dnsprovider.com` | Delegation: "these servers are authoritative" |
| **TXT** | Domain → free text | `"v=spf1 include:_spf.google.com ~all"` | Domain ownership proof, SPF/DKIM email security |

A few notes:

- **CNAME chains:** `www.shop.com → shop.cdn-provider.net → 151.101.1.1`. The resolver follows the chain until it hits an A/AAAA record.
- **CNAME can't be at the root** of a domain (`shop.com` itself) by the DNS spec. Many providers offer a workaround called **ALIAS / ANAME / CNAME flattening**.
- Other records you may hear about: **SOA** (zone metadata), **SRV** (service host + port), **PTR** (reverse lookup: IP → name), **CAA** (which CAs may issue certificates).

---

## TTL: how long answers are trusted

Every record comes with a **TTL in seconds**:

```
shop.com.   300   IN   A   93.184.216.34
            ^^^
            cache this for 5 minutes
```

| TTL | Pros | Cons |
|---|---|---|
| **Short** (30-300 s) | Changes (failover, migration) take effect fast | More queries, slightly slower lookups, more load on DNS |
| **Long** (hours-days) | Fewer lookups, faster, resilient if DNS provider blips | Changes take a long time to reach everyone |

**Migration trick:** planning to move servers? Lower the TTL (say to 60 s) a day or two *before* the move. Once the old long TTL has expired everywhere, switch the IP — the change now propagates in about a minute. Raise the TTL again afterward.

> "DNS propagation takes 24-48 hours" really means "old answers sit in caches until their TTL expires." Some resolvers and clients also ignore TTLs and cache longer, which is why changes can feel slow.

---

## Caching layers

A lookup stops at the **first layer that has a fresh answer**:

```
┌──────────────────┐  hit? → done (0 ms)
│ Browser cache    │
├──────────────────┤
│ OS cache         │  hit? → done (also checks /etc/hosts)
├──────────────────┤
│ Router cache     │  hit? → done (home routers often cache)
├──────────────────┤
│ Recursive        │  hit? → done (~5-30 ms)  ← shared by millions of users,
│ resolver cache   │                            so popular names are almost always here
├──────────────────┤
│ Root → TLD →     │  full lookup (~50-200 ms)
│ Authoritative    │
└──────────────────┘
```

Resolvers also cache the *intermediate* answers (e.g. "`.com` TLD servers are here"), so even a "full" lookup rarely goes all the way to the root.

---

## DNS-based load balancing

Because DNS chooses which IP a client gets, you can spread traffic with it.

### Round-robin DNS

Return several A records; clients generally pick one, and the order rotates.

```
shop.com  A  10.0.0.1
shop.com  A  10.0.0.2
shop.com  A  10.0.0.3
```

✅ Simple, free.
❌ No real health checks in plain DNS — a dead server keeps getting traffic until you remove it *and* caches expire. ❌ Clients cache, so the spread is uneven.

### Geo-DNS (location-based routing)

The authoritative server looks at where the query comes from and returns the **nearest data center**:

```
User in Mumbai   ──► DNS ──► "13.235.x.x"   (Mumbai region)
User in London   ──► DNS ──► "18.130.x.x"   (London region)
User in Virginia ──► DNS ──► "54.85.x.x"    (US-East region)
```

This is how global apps and CDNs send users to a nearby server. Managed DNS services (AWS Route 53, Cloudflare, NS1) also offer:

| Policy | Idea |
|---|---|
| **Latency-based** | Return the region with the lowest measured latency to the user |
| **Weighted** | Send 90% to v1, 10% to v2 (canary releases) |
| **Failover** | Health-check the primary; if it's down, answer with the backup's IP |

### Limitation to remember

DNS sees the **resolver's** location, not the user's. A user in Delhi using a resolver in Singapore may get routed to Singapore. The **EDNS Client Subnet** extension helps by passing part of the user's IP along.

DNS load balancing is usually the **first, coarse layer** (pick a region), with a real load balancer inside each region doing the fine-grained work.
