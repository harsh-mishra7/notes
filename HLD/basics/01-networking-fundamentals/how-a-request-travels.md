# How a Request Travels

## TL;DR

You type `https://shop.com/products` and press Enter. In the next few hundred milliseconds, your browser:

1. **Finds the server's address** (DNS)
2. **Opens a connection** to it (TCP handshake)
3. **Makes that connection private** (TLS handshake)
4. **Asks for the page** (HTTP request)
5. The server **does its work** (load balancer → app → database)
6. The server **sends the page back** (HTTP response)
7. The browser **turns the bytes into pixels** (rendering)

Every system design topic hooks into one of these steps. Learn this flow once, and the rest of HLD has a place to live.

---

## The analogy that makes it click

**Sending a request is like ordering from a restaurant by phone.**

| Step | Restaurant version | Web version |
|---|---|---|
| Look up the number | Search "Pizza Place" in your contacts | **DNS**: `shop.com` → `93.184.216.34` |
| Dial and hear "Hello?" | Call connects, both sides confirm they can hear | **TCP handshake** |
| Agree on a secret code | "Let's speak in a code only we know" | **TLS handshake** |
| Place the order | "One large margherita, please" | **HTTP request** |
| Kitchen cooks | Chef makes the pizza | **Server processing** |
| Delivery arrives | Pizza at your door | **HTTP response** |
| You plate and eat it | Arrange it on the table | **Rendering** |

You wouldn't dial before knowing the number, and you wouldn't order before the call connects. The web works the same way — each step depends on the one before.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **URL** | The full address: `https://shop.com:443/products?id=7` |
| **DNS** | The internet's phone book. Turns names into IP addresses. |
| **IP address** | The numeric address of a machine, like `93.184.216.34` |
| **RTT** (Round Trip Time) | Time for a message to go there and back. The unit of "slow". |
| **Handshake** | A few back-and-forth messages to set up a connection before real data flows |

---

## The big picture

```
 YOU                                                         SERVER SIDE
┌────────┐
│Browser │
└───┬────┘
    │ 1. DNS: "Where is shop.com?"
    ├──────────────────────────► DNS resolver ──► "93.184.216.34"
    │
    │ 2. TCP handshake  (SYN → SYN-ACK → ACK)
    ├──────────────────────────────────────────►┌──────────────┐
    │                                            │ Load balancer│
    │ 3. TLS handshake  (certs + keys)           │  / CDN edge  │
    ├──────────────────────────────────────────►└──────┬───────┘
    │                                                   │
    │ 4. HTTP request:  GET /products                   ▼
    ├──────────────────────────────────────────► ┌─────────────┐
    │                                            │ App server  │
    │                              5. Processing │   │         │
    │                                            │   ▼         │
    │                                            │ Cache / DB  │
    │ 6. HTTP response: 200 OK + HTML            └─────────────┘
    │◄──────────────────────────────────────────
    │
    │ 7. Parse HTML → fetch CSS/JS/images → paint pixels
    ▼
  Page on screen
```

Now let's walk through each step.

---

## Step 0: Parse the URL

Before any network traffic, the browser breaks the URL into parts:

```
https :// shop.com : 443 /products ? id=7
  │         │        │      │         │
scheme    host     port   path     query
```

- `https` → use TLS, default port **443** (`http` would be port 80)
- `shop.com` → the name we need to look up

It also checks: is this in my **browser cache** already? If yes and still fresh, it may skip the network entirely.

---

## Step 1: DNS lookup — find the address

Computers route by IP address, not names. So the browser asks: "What's the IP for `shop.com`?"

It checks caches in order, stopping at the first hit:

```
Browser cache → OS cache → Router → ISP's DNS resolver → (full lookup: root → .com → shop.com's nameserver)
```

Result: `shop.com → 93.184.216.34`

- **Cached:** ~0-few ms
- **Full lookup:** ~20-100+ ms

> Deep dive: see `dns.md`.

---

## Step 2: TCP handshake — open a reliable connection

Now the browser knows *where* to go. It opens a **TCP connection** to `93.184.216.34:443`.

```
Browser                          Server
   │ ──── SYN ─────────────────►  │   "Can we talk?"
   │ ◄─── SYN-ACK ─────────────   │   "Yes, can you hear me?"
   │ ──── ACK ─────────────────►  │   "Yes. Let's go."
   │                              │
   │    connection established    │
```

**Cost: 1 RTT** before any data can be sent.

> Deep dive: see `ip-ports-tcp-udp.md`.

---

## Step 3: TLS handshake — make it private

Because it's `https`, the browser and server now agree on encryption keys and the browser verifies the server is really `shop.com`.

```
Browser                               Server
   │ ── ClientHello (ciphers, key share) ─►│
   │ ◄─ ServerHello + certificate + key ── │
   │    (browser verifies certificate)     │
   │ ── Finished ────────────────────────► │
   │                                       │
   │       encrypted channel ready         │
```

- **TLS 1.3:** 1 RTT
- **TLS 1.2 (older):** 2 RTTs

> Deep dive: see `http-https.md`.

---

## Step 4: Send the HTTP request

Finally, the actual question:

```http
GET /products?id=7 HTTP/1.1
Host: shop.com
User-Agent: Mozilla/5.0 ...
Accept: text/html
Cookie: session=abc123
```

It says: *which action* (`GET`), *which resource* (`/products?id=7`), and *extra context* in headers (who I am, what formats I accept).

---

## Step 5: Server processing — where HLD lives

This is the part system design interviews care most about. A real request often passes through many layers:

```
Internet
   │
   ▼
┌─────────┐   ┌───────────────┐   ┌──────────────┐   ┌─────────┐
│ CDN /   │──►│ Load balancer │──►│  App server  │──►│  Cache  │
│ WAF     │   │ (picks a box) │   │ (your code)  │   │ (Redis) │
└─────────┘   └───────────────┘   └──────┬───────┘   └────┬────┘
                                         │   miss          │
                                         ▼                 │
                                   ┌──────────┐            │
                                   │ Database │◄───────────┘
                                   └──────────┘
```

| Layer | Job |
|---|---|
| **CDN** | Serve cached static files close to the user |
| **Load balancer** | Spread requests across many app servers |
| **App server** | Run business logic, check auth, build the response |
| **Cache** | Return hot data fast without hitting the DB |
| **Database** | The source of truth |

Each layer is a topic of its own in HLD.

---

## Step 6: The HTTP response comes back

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 5123
Cache-Control: max-age=60
Set-Cookie: session=abc123

<!DOCTYPE html><html>...
```

- **Status code** (`200`) says how it went
- **Headers** describe the body and caching rules
- **Body** is the HTML itself

The connection usually stays open (**keep-alive**) so the next requests skip steps 2 and 3.

---

## Step 7: Rendering — bytes to pixels

The browser now turns HTML into a visible page:

```
HTML ──► DOM tree ─┐
                   ├──► Render tree ──► Layout ──► Paint ──► Composite ──► Screen
CSS  ──► CSSOM  ───┘       (what to show)  (where)   (colors)   (layers)
```

While parsing HTML, it finds more resources (`<link>`, `<script>`, `<img>`) and fetches them — **each one repeats this flow** (though DNS/TCP/TLS are usually reused).

A typical page makes **dozens of requests**. That's why latency per request matters so much.

---

## Where the time goes

A rough budget for a first visit to a server ~50 ms away (RTT = 100 ms):

| Step | Cost | Can be skipped when… |
|---|---|---|
| DNS | 0-100 ms | Already cached |
| TCP handshake | 1 RTT (100 ms) | Connection reused (keep-alive) |
| TLS handshake | 1 RTT (100 ms, TLS 1.3) | Connection reused / session resumption |
| Request + response | 1 RTT + server time | Response cached (browser/CDN) |
| Rendering | 10s-100s of ms | — |

**~300+ ms before the first byte arrives** — and most of it is *waiting*, not computing. This is why CDNs (shorter RTT), connection reuse, and HTTP/3 (fewer handshakes) exist.

---

## Where this shows up in HLD

- **"What happens when you type a URL?"** is a classic interview warm-up. It tests whether you see the whole path.
- **Latency discussions:** knowing handshakes cost RTTs explains *why* CDNs, edge servers, and connection pooling help.
- **Placing components:** the server-side layers (CDN → LB → app → cache → DB) are the skeleton of almost every HLD diagram you'll draw.
- **Failure points:** DNS down, cert expired, LB misrouting, DB slow — each step is a place things break, and a place to add redundancy.
- **Optimization:** every improvement you'll propose (caching, CDN, HTTP/2, keep-alive) removes or shortens one of these steps.

---

## Key takeaways

- A request goes **DNS → TCP → TLS → HTTP request → server → HTTP response → render**.
- Handshakes cost **round trips**; distance and round trips, not CPU, dominate web latency.
- **Caching and connection reuse** let later requests skip most of these steps.
- The **server-side path** (CDN, load balancer, app, cache, DB) is where system design happens.
- One page load = **dozens of requests**, so small per-request costs multiply.
