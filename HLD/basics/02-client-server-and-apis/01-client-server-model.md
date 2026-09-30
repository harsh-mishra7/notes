# Client-Server Model

## Brief

**One side asks, the other side answers.**

A **client** is anything that *asks* for something: a browser, a mobile app, another program.
A **server** is a program that *waits* for those requests and sends back a **response**.

Almost every system you'll design in HLD, from a to-do app to Netflix, is this same pattern repeated and stacked in layers.

---

## The analogy that makes it click

**A restaurant.**

- **You (the client)** sit at a table and order from the menu.
- **The waiter (the server / backend)** takes your order to the kitchen and brings the food back.
- **The kitchen's pantry (the database)** holds all the ingredients. You never walk into it yourself.

A few things hold in both a restaurant and a real system:

- You can only order what's **on the menu**. That menu is the **API**.
- The waiter serves **many tables at once**. One server handles many clients.
- You never touch the pantry directly. Clients never talk to the database directly.
- If it gets busy, the restaurant **hires more waiters**. That's scaling.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Client** | The program that sends a request (browser, app, CLI, another service) |
| **Server** | The program that listens for requests and returns responses |
| **Request** | A message from client to server: "give me X" or "do Y" |
| **Response** | The server's reply: the data, plus a status (worked / failed) |
| **Protocol** | The agreed-upon language they speak (usually HTTP) |
| **API** | The list of requests a server accepts, i.e. its "menu" |

> "Server" means two things: the **program** (e.g. a Node.js app) and the **machine** it runs on. Context usually tells you which one.

---

## The request/response cycle

Here's what happens when you open `https://shop.com/products/42`:

```
  CLIENT (browser)                                   SERVER
       │                                                │
  1.   │── DNS: "what's the IP of shop.com?" ──► DNS    │
       │◄── "93.184.216.34" ────────────────────        │
       │                                                │
  2.   │── open TCP connection (+ TLS handshake) ──────►│
       │                                                │
  3.   │── GET /products/42 ───────────────────────────►│
       │                                                │  4. run code,
       │                                                │     query database
       │                                                │
  5.   │◄── 200 OK  { "id": 42, "name": "Shoes" } ──────│
       │                                                │
  6.   render the page
```

In words:

1. **Find the server.** DNS turns the domain name into an IP address.
2. **Connect.** Open a TCP connection (and a TLS handshake for HTTPS).
3. **Send the request.** Method + path + headers + optional body.
4. **Server does work.** Checks who you are, reads/writes the database, builds a reply.
5. **Send the response.** Status code + headers + body (often JSON or HTML).
6. **Client uses it.** Renders a page, updates the UI, etc.

A raw HTTP request and response look like this:

```http
GET /products/42 HTTP/1.1
Host: shop.com
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": 42, "name": "Shoes", "price": 59.99 }
```

**Key point:** the client always starts the conversation. In plain HTTP the server can't "call" the client out of the blue. (For live updates you need extra tools like WebSockets, Server-Sent Events, or polling.)

---

## Thin client vs thick client

How much work happens *on the client* vs *on the server*?

```
THIN CLIENT                              THICK (FAT) CLIENT

┌──────────┐                             ┌──────────────────────┐
│ Browser  │  just displays what         │ React / mobile app   │
│ (dumb)   │  the server sends           │ - UI logic           │
└────┬─────┘                             │ - routing            │
     │  "here's the full HTML page"      │ - validation, cache  │
┌────▼──────────────────┐                └──────────┬───────────┘
│ Server does it all:   │                           │  "just give me JSON"
│ logic + HTML render   │                ┌──────────▼───────────┐
└───────────────────────┘                │ Server: data + rules │
                                         └──────────────────────┘
```

| | Thin client | Thick client |
|---|---|---|
| **Where logic lives** | Mostly on the server | A lot on the client |
| **Examples** | Server-rendered pages (classic PHP, Rails views), terminals | SPAs (React, Angular), mobile apps, desktop apps, games |
| **Needs a strong device?** | No | More so |
| **Works offline?** | ❌ | Can, partially ✅ |
| **Updating** | Deploy the server, everyone gets it | Mobile/desktop users must update the app |
| **Server load** | Higher (it does more) | Lower per request |

**Important:** even with a thick client, **never trust the client**. Anything on the user's device can be modified. Security checks, prices, and permissions must always be validated on the server.

---

## Tiers: how the pieces are split up

A **tier** is a separate layer, usually running on separate machines.

### 1-tier

Everything on one machine: UI, logic, and data. Think a desktop app with a local file, or Excel. Fine for personal tools; not a "system design" topic really.

### 2-tier

The client talks **directly** to the database.

```
┌──────────┐            ┌────────────┐
│  Client  │ ─────────► │  Database  │
│ (UI +    │   SQL      └────────────┘
│  logic)  │
└──────────┘
```

Seen in old desktop business apps. The problems:

- ❌ Every client needs DB credentials (a security nightmare).
- ❌ Business logic is copied into every client; changing it means updating every install.
- ❌ Databases can only handle so many direct connections.

### 3-tier (the standard)

Put a server **in the middle**.

```
┌──────────────┐        ┌──────────────────┐        ┌──────────────┐
│ PRESENTATION │  HTTP  │   APPLICATION    │  SQL   │     DATA     │
│   (client)   │ ─────► │    (backend)     │ ─────► │  (database)  │
│              │ ◄───── │                  │ ◄───── │              │
│ React, iOS,  │  JSON  │ Node, Java, Go,  │        │ Postgres,    │
│ Android      │        │ Python API       │        │ MySQL, Mongo │
└──────────────┘        └──────────────────┘        └──────────────┘
    FRONTEND                 BACKEND                   STORAGE
```

| Tier | Job | Who can touch it |
|---|---|---|
| **Presentation** | Show things, collect input | The user |
| **Application** | Business rules, auth, validation | Only clients, via the API |
| **Data** | Store and retrieve data | Only the application tier |

Why it wins:

- ✅ The database is hidden. Only the backend has credentials.
- ✅ Business logic lives in one place.
- ✅ Each tier scales on its own (add more backend servers without touching the DB).
- ✅ You can have many clients (web, iOS, Android) sharing one backend.

### N-tier (what real systems look like)

Real systems keep adding layers in between:

```
Client ─► CDN ─► Load Balancer ─► API servers ─► Cache (Redis)
                                        │
                                        ├─► Database
                                        ├─► Message queue ─► Workers
                                        └─► Other services
```

Each box is still just a client-server pair. **The API server is a server to the browser, and a client to the database.** Roles depend on who's asking whom.

---

## Where the frontend, backend, and DB actually sit

| Part | Runs where | Examples |
|---|---|---|
| **Frontend** | On the user's device (browser/phone). The *files* are often served from a CDN. | React, Vue, Swift, Kotlin |
| **Backend** | On your servers / cloud (VMs, containers, serverless) | Node/Express, Spring, Django, Go |
| **Database** | On your servers, in a private network, not reachable from the internet | Postgres, MySQL, MongoDB, DynamoDB |

A common confusion: the React code is *downloaded from* a server, but it *runs on* the user's machine. That's why it's the client.

---

## Client-server vs peer-to-peer

| | Client-server | Peer-to-peer (P2P) |
|---|---|---|
| **Who serves** | Dedicated servers | Every participant is both client and server |
| **Control** | Central, easy to secure and update | Decentralized, harder to control |
| **Single point of failure** | Yes (unless you add redundancy) | No |
| **Examples** | Websites, most apps | BitTorrent, some video calls (WebRTC), blockchains |

Most HLD interviews are about client-server systems.

---

## Gotchas

- **The server is a bottleneck and a single point of failure.** One server = one crash away from downtime. This is why load balancers and replicas exist.
- **Network calls are slow and can fail.** Every hop adds latency and a chance of timeout. Design for retries and timeouts.
- **Never trust client input.** Validate everything on the server.
- **Don't expose the database to the internet.** Always go through the application tier.
