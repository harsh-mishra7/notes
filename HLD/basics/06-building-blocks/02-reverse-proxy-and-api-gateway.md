# Reverse Proxy and API Gateway

## TL;DR

**A proxy is a middleman that passes requests along on someone's behalf.**

- A **forward proxy** acts on behalf of **clients** (hides the users from the internet).
- A **reverse proxy** acts on behalf of **servers** (hides the servers from the users).
- An **API gateway** is a reverse proxy with extra, API-specific smarts: routing to microservices, authentication, rate limiting, and combining responses.

A load balancer, a reverse proxy, and an API gateway overlap a lot. Often the *same software* (like Nginx or Envoy) does all three jobs.

---

## The analogy that makes it click

**Think of a big company's front desk.**

- **Forward proxy = your personal assistant.** You want to call someone outside the company, so your assistant places the call for you. The other person only sees the assistant's number, not yours.
- **Reverse proxy = the company receptionist.** Outsiders call one main number. The receptionist answers, figures out who they need, and transfers the call. Callers never learn the internal extensions.
- **API gateway = a receptionist who also checks ID badges.** They verify who you are, make sure you're not calling 500 times a minute, route you to the right department, and can collect answers from three departments and hand them to you in one go.

---

## Forward proxy vs reverse proxy

```
FORWARD PROXY  (sits with the clients)

  [You] ─┐
  [You] ─┼──► [ Forward Proxy ] ──────► Internet  ──► google.com
  [You] ─┘
             google.com sees the PROXY's IP, not yours


REVERSE PROXY  (sits with the servers)

                                          ┌──► [ server A ]
  Internet users ──► [ Reverse Proxy ] ───┼──► [ server B ]
                                          └──► [ server C ]
             users see only the PROXY; servers stay hidden
```

| | **Forward proxy** | **Reverse proxy** |
|---|---|---|
| Works for | Clients | Servers |
| Who is hidden | The client | The server |
| Who sets it up | Client side (company network, user) | Server side (the website owner) |
| Typical uses | Corporate web filtering, bypassing geo-blocks, VPN-like privacy, caching for an office | TLS termination, load balancing, caching, security |
| Examples | Squid, corporate proxies | Nginx, HAProxy, Envoy, Cloudflare |

In system design interviews, **reverse proxy** is the one that matters. Forward proxies are mostly a networking/security topic.

---

## What a reverse proxy does

Once all traffic flows through one point, that point can do a lot of useful work so your app servers don't have to.

| Job | What it means |
|---|---|
| **TLS termination** | The proxy handles HTTPS (decrypting/encrypting). Your app servers speak plain HTTP on the private network. One place to manage certificates. |
| **Load balancing** | Spreads requests across several backend servers. |
| **Caching** | Stores responses (e.g. a popular product page) and serves them without hitting the backend. |
| **Compression** | gzip/brotli responses before sending, saving bandwidth. |
| **Hiding servers** | Clients never learn backend IPs or how many servers exist. Smaller attack surface. |
| **Security filtering** | Block bad IPs, limit request size, act as a basic firewall (WAF). |
| **Serving static files** | Nginx serves images/CSS directly, much faster than your app framework. |
| **Buffering slow clients** | Proxy absorbs a slow mobile upload, then hands it to the app in one fast burst, so app workers aren't tied up. |

```
           HTTPS                    plain HTTP (private network)
 Client ═══════════► [ Reverse Proxy ] ─────────────► App server
                       - decrypts TLS
                       - checks cache  → HIT? reply immediately
                       - compresses the response
                       - picks a backend
```

---

## API Gateway

In a **microservices** system, the "backend" isn't one app — it's dozens of small services. Clients shouldn't need to know about all of them.

```
WITHOUT a gateway                      WITH a gateway

 Mobile app ──► user-service           Mobile app ──► [ API Gateway ] ─┬─► user-service
            ──► order-service                                          ├─► order-service
            ──► payment-service                                        ├─► payment-service
            ──► review-service                                         └─► review-service

 Client knows every service,           Client knows ONE URL.
 each one re-implements auth.           Auth, limits, logging done once.
```

### What an API gateway adds

| Feature | What it means |
|---|---|
| **Routing** | `/users/*` → user-service, `/orders/*` → order-service |
| **Authentication** | Verify the API key / JWT / OAuth token once, at the door. Services behind trust the gateway. |
| **Rate limiting** | "Max 100 requests per minute per API key." Stops abuse before it reaches services. |
| **Request aggregation** | One client call → gateway calls 3 services → merges into one response. Saves mobile round trips. |
| **Protocol translation** | Client speaks REST/JSON; internally services speak gRPC. |
| **Request/response transformation** | Add headers, rename fields, strip internal data. |
| **Monitoring & logging** | One place to record every API call, latency, and error rate. |
| **Versioning & canary** | Send `/v2/*` or 5% of traffic to a new service version. |

### Request aggregation example

```
Mobile app:  GET /home-screen
                   │
            [ API Gateway ]
         ┌─────────┼──────────┐
         ▼         ▼          ▼
   user-service  orders    recommendations
   (profile)     (recent)  (for you)
         └─────────┼──────────┘
                   ▼
    One combined JSON response  ← 1 round trip for the phone instead of 3
```

A related pattern is **BFF (Backend for Frontend)**: a separate gateway per client type (one for web, one for mobile), each shaped for that client's needs.

### Downsides of an API gateway

- **Another hop** — adds a little latency to every request.
- **Single point of failure** — must be run redundantly, like a load balancer.
- **Can become a bottleneck or a "god service"** if you stuff too much business logic into it. Keep it about cross-cutting concerns (auth, limits, routing), not business rules.

---

## How LB, reverse proxy, and API gateway relate

They're not three separate boxes so much as **three levels of smartness** on the same idea: "one entry point in front of servers."

```
 ┌───────────────────────────────────────────────────┐
 │  API Gateway                                      │
 │  + auth, rate limiting, aggregation, API keys     │
 │  ┌─────────────────────────────────────────────┐  │
 │  │  Reverse Proxy                              │  │
 │  │  + TLS termination, caching, compression,   │  │
 │  │    hiding servers                           │  │
 │  │  ┌───────────────────────────────────────┐  │  │
 │  │  │  Load Balancer                        │  │  │
 │  │  │  spread traffic across servers,       │  │  │
 │  │  │  health checks                        │  │  │
 │  │  └───────────────────────────────────────┘  │  │
 │  └─────────────────────────────────────────────┘  │
 └───────────────────────────────────────────────────┘
```

- Every **load balancer** (L7) is technically a kind of reverse proxy.
- A **reverse proxy** often load balances, but can also sit in front of just *one* server (for TLS, caching, security).
- An **API gateway** is a reverse proxy specialized for APIs.

### Comparison table

| | **Load Balancer** | **Reverse Proxy** | **API Gateway** |
|---|---|---|---|
| Main goal | Spread load, availability | Protect & offload servers | Manage APIs for many services |
| Needs multiple backends? | Yes (that's the point) | No — useful even with one | Usually many services |
| Routing | By algorithm (RR, least conn) | By host/path | By path, method, version, API key |
| TLS termination | Sometimes (L7) | ✅ Yes | ✅ Yes |
| Caching / compression | Rarely | ✅ Yes | Sometimes |
| Authentication | ❌ No | Basic at most | ✅ Yes (JWT, OAuth, API keys) |
| Rate limiting | ❌ Rarely | Basic | ✅ Yes, per user/key |
| Request aggregation | ❌ No | ❌ No | ✅ Yes |
| Examples | AWS ALB/NLB, HAProxy | Nginx, HAProxy, Envoy, Cloudflare | Kong, AWS API Gateway, Apigee, Envoy-based gateways |

### A typical real setup

```
Client ──► CDN ──► Load Balancer ──► API Gateway ──► Microservices
                    (L4/L7, cloud)    (auth, limits,    (each may have
                                       routing)          its own LB)
```

In smaller systems, a single Nginx instance can be the reverse proxy, load balancer, *and* do basic gateway work.

---

## Where this shows up in HLD

- **Microservices designs** almost always get an API gateway at the front. Mention it when you split a system into services.
- **"Where do you do authentication and rate limiting?"** — the gateway is the standard answer, so each service doesn't re-implement it.
- **TLS termination** at the proxy/LB is the usual answer to "where does HTTPS end?"
- **Mobile clients** benefit from gateway aggregation (fewer round trips on slow networks).
- Interviewers like hearing that you know these **overlap** — don't draw three separate boxes if one Nginx would do; do separate them when scale or team boundaries call for it.

---

## Key takeaways

- A **forward proxy** hides clients; a **reverse proxy** hides servers. System design cares about the reverse proxy.
- Reverse proxies handle TLS termination, caching, compression, and security so app servers stay simple.
- An **API gateway** is a reverse proxy for APIs: routing to microservices, auth, rate limiting, and request aggregation.
- LB, reverse proxy, and API gateway overlap heavily — often one tool (Nginx, Envoy) plays several roles.
- Every entry point is a potential single point of failure, so run it redundantly — and keep business logic out of the gateway.
