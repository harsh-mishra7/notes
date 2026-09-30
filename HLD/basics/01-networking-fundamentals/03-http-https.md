# HTTP and HTTPS

## Brief

**HTTP** is the language browsers and servers speak. It's a simple **request → response** conversation:

- The client says *what it wants* (a **method** like `GET`, a **path** like `/users/42`, and some **headers**).
- The server answers with *how it went* (a **status code** like `200` or `404`), headers, and a body.

**HTTPS** is the same HTTP, wrapped in **TLS encryption** so nobody in between can read or tamper with it.

HTTP has evolved: **HTTP/1.1** (one request at a time per connection) → **HTTP/2** (many requests in parallel over one connection) → **HTTP/3** (same idea, but over **QUIC/UDP** to avoid TCP's bottlenecks).

---

## The analogy that makes it click

**HTTP is a counter at a government office.**

- You fill in a form: *type of request* (method), *which department* (path), *your details* (headers), *attachments* (body).
- The clerk stamps a result on it: **Approved** (2xx), **Go to another counter** (3xx), **Your form is wrong** (4xx), **Our system is down** (5xx).
- The clerk doesn't remember you between visits — you show your ID (cookie/token) every time. HTTP is **stateless**.

**HTTPS** is the same office, but you talk through a sealed tube only you and the clerk can open — and the clerk shows a government-issued badge (certificate) proving it's really the office.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Method** | The action: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`... |
| **Status code** | 3-digit result of the request |
| **Header** | `Key: value` metadata about the request or response |
| **Body** | The actual content (HTML, JSON, image...) |
| **Stateless** | Each request stands alone; the server doesn't remember previous ones by default |
| **TLS** | Transport Layer Security. The encryption behind HTTPS. |

---

## What a request and response look like

```http
POST /api/orders HTTP/1.1          ← method, path, version
Host: shop.com                     ← headers
Content-Type: application/json
Authorization: Bearer eyJhbGci...

{"productId": 7, "qty": 2}         ← body
```

```http
HTTP/1.1 201 Created               ← version, status
Content-Type: application/json     ← headers
Location: /api/orders/991

{"id": 991, "status": "placed"}    ← body
```

It's just structured text (in HTTP/1.1). That's part of why it took over the world.

---

## Methods

| Method | Purpose | Has body? | Safe? | Idempotent? |
|---|---|---|---|---|
| `GET` | Read a resource | No | ✅ | ✅ |
| `POST` | Create / trigger an action | Yes | ❌ | ❌ |
| `PUT` | Replace a resource entirely | Yes | ❌ | ✅ |
| `PATCH` | Partially update a resource | Yes | ❌ | ❌ (usually) |
| `DELETE` | Remove a resource | Optional | ❌ | ✅ |
| `HEAD` | Like GET, headers only | No | ✅ | ✅ |
| `OPTIONS` | "What's allowed here?" (used in CORS preflight) | No | ✅ | ✅ |

- **Safe** = doesn't change anything on the server.
- **Idempotent** = doing it 1 time or 10 times leaves the server in the same state. `DELETE /users/42` twice → user is still just deleted. `POST /orders` twice → **two orders**.

Idempotency matters a lot for **retries**: you can safely retry idempotent requests after a timeout. For `POST`, systems use an **idempotency key** header so a retried payment isn't charged twice.

---

## Status codes (grouped)

| Range | Meaning | Common ones |
|---|---|---|
| **1xx** | Informational — "keep going" | `101 Switching Protocols` (WebSocket upgrade) |
| **2xx** | Success | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | Redirect — "look elsewhere" | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| **4xx** | Client error — "you did something wrong" | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests` |
| **5xx** | Server error — "we did something wrong" | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

A few that confuse people:

| Code | Actually means |
|---|---|
| `401` | "I don't know who you are" (not logged in / bad token) |
| `403` | "I know who you are, and you're not allowed" |
| `304` | "Your cached copy is still good, use it" (no body sent) |
| `502` | A proxy/LB got a *bad* response from the server behind it |
| `504` | A proxy/LB got *no* response in time from the server behind it |
| `429` | Rate limited — slow down |

**Retry rule of thumb:** 5xx and 429 are often worth retrying (with backoff). 4xx usually aren't — the request itself is wrong.

---

## Headers you'll meet constantly

| Header | Direction | Purpose |
|---|---|---|
| `Host` | Request | Which site (one IP can host many sites) |
| `Content-Type` | Both | Format of the body: `application/json`, `text/html` |
| `Authorization` | Request | Credentials, e.g. `Bearer <token>` |
| `Cookie` / `Set-Cookie` | Req / Resp | Send / store small state (sessions) |
| `Cache-Control` | Both | Caching rules (`max-age`, `no-store`...) |
| `ETag` / `If-None-Match` | Resp / Req | Version tag for conditional requests → `304` |
| `Accept-Encoding` / `Content-Encoding` | Req / Resp | Compression (`gzip`, `br`) |
| `Location` | Response | Where to go (redirects, newly created resources) |
| `User-Agent` | Request | What client is making the request |

---

## Keep-alive: reuse the connection

Opening a connection costs a TCP handshake (+ TLS). Doing that for every image would be painfully slow.

```
WITHOUT keep-alive (HTTP/1.0 default)     WITH keep-alive (HTTP/1.1 default)

[handshake] GET /     [close]             [handshake] GET /
[handshake] GET /a.css [close]                        GET /a.css
[handshake] GET /b.js  [close]                        GET /b.js
                                                      ...  [close when idle]
```

HTTP/1.1 keeps connections open by default (`Connection: keep-alive`). The same idea on the server side is **connection pooling** (e.g. app → database).

---

## HTTPS and TLS basics

Plain HTTP is readable by anyone between you and the server (public Wi-Fi, ISP, compromised router). TLS gives three things:

| Property | Meaning |
|---|---|
| **Encryption** | Nobody in the middle can read the data |
| **Integrity** | Nobody can modify it without detection |
| **Authentication** | You're really talking to `shop.com`, not an impostor |

### How the handshake works (TLS 1.3, simplified)

```
Browser                                            Server
   │ ── ClientHello: supported ciphers + key share ──►│
   │                                                  │
   │ ◄── ServerHello: chosen cipher + key share ───── │
   │     + Certificate (signed by a trusted CA)       │
   │     + Finished                                   │
   │                                                  │
   │  Browser checks certificate against trusted CAs  │
   │  Both sides derive the same session key          │
   │                                                  │
   │ ── Finished + first encrypted request ─────────► │
```

- **Asymmetric crypto** (public/private keys) is used only to agree on a key and prove identity.
- **Symmetric crypto** (fast, one shared key) encrypts the actual data.
- The **certificate** is issued by a **Certificate Authority** (CA) like Let's Encrypt. Your browser/OS ships with a list of CAs it trusts.
- TLS 1.3 = **1 RTT** handshake (0-RTT possible on resumption). TLS 1.2 = 2 RTTs.

**TLS termination:** in most systems, the load balancer or CDN handles TLS and forwards plain HTTP (or re-encrypted traffic) to internal servers. This centralizes certificates and saves CPU on app servers.

---

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### The problem each version fixes

**HTTP/1.1:** one connection handles **one request at a time**. Browsers work around it by opening ~6 connections per domain.

```
Conn 1: [ GET a ]──────[ GET d ]──────
Conn 2: [ GET b ]──────[ GET e ]──────      max ~6, requests queue
Conn 3: [ GET c ]──────[ GET f ]──────
```

**HTTP/2:** **multiplexing** — many requests interleaved as small frames on **one** TCP connection.

```
One TCP conn:  a1 b1 c1 a2 d1 b2 c2 a3 ...   all in flight at once
```

But if one TCP packet is lost, TCP holds back *everything* behind it — all streams stall. That's **TCP head-of-line blocking**.

**HTTP/3:** runs on **QUIC**, built on **UDP**. QUIC does its own reliability *per stream*, so a lost packet only stalls the stream it belongs to. It also merges the transport and TLS handshakes.

```
HTTP/1.1, HTTP/2               HTTP/3
┌──────────┐                   ┌──────────┐
│   HTTP   │                   │  HTTP/3  │
├──────────┤                   ├──────────┤
│   TLS    │                   │   QUIC   │ ← reliability + TLS 1.3 built in
├──────────┤                   ├──────────┤
│   TCP    │                   │   UDP    │
└──────────┘                   └──────────┘
```

### Comparison

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Year | 1997 | 2015 | 2022 |
| Transport | TCP | TCP | QUIC (over UDP) |
| Format | Text | Binary frames | Binary frames |
| Requests per connection | One at a time | Many in parallel (multiplexed) | Many in parallel |
| Header compression | ❌ | ✅ HPACK | ✅ QPACK |
| Head-of-line blocking | At HTTP level | At TCP level | ✅ Mostly solved |
| Setup cost (new conn) | TCP + TLS (2-3 RTT) | TCP + TLS (2-3 RTT) | 1 RTT (0-RTT on resume) |
| Survives network change (Wi-Fi → 4G) | ❌ | ❌ | ✅ Connection IDs |
| Encryption | Optional | Effectively required by browsers | Always (built in) |

HTTP/2 and HTTP/3 keep the same methods, status codes, and headers — only the transport underneath changes. Your API code doesn't care which version is used.
