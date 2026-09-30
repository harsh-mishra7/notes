# API Styles: REST, GraphQL, gRPC

## TL;DR

An **API** is the menu a server offers: the list of requests it accepts and what it sends back.

There are three popular ways to design that menu:

- **REST**: many URLs, one per "thing" (resource). Uses plain HTTP verbs. The default for public web APIs.
- **GraphQL**: one URL. The client sends a query describing *exactly* the data it wants.
- **gRPC**: call functions on another server as if they were local. Binary, fast, strict. Mostly for service-to-service traffic inside a company.

None is "best". Each fits a different job.

---

## The analogy that makes it click

**Three ways to get food.**

- **REST = a set menu.** Dish #3 always comes with rice, salad, and soup. You get what's on the plate, even if you only wanted the rice. Need dessert too? That's a second order.
- **GraphQL = a buffet with a custom order slip.** You write "rice, no salad, and one dessert" on one slip and get exactly that in one trip.
- **gRPC = the kitchen's internal intercom.** Chefs shout short coded orders to each other ("#7, two, rush"). Super fast and precise, but customers don't use it.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Endpoint** | A URL the server listens on, e.g. `/users/42` |
| **Resource** | A "thing" your API exposes: a user, an order, a product |
| **Payload** | The data in the body of a request or response |
| **Schema** | A formal description of what data looks like (fields and types) |
| **Over-fetching** | Getting more data than you need |
| **Under-fetching** | Not getting enough in one call, so you make several |
| **Serialization** | Turning data into bytes to send over the wire (JSON, protobuf) |

---

## REST

**REST** (Representational State Transfer) models your system as **resources** (nouns) and uses **HTTP methods** (verbs) to act on them.

### Resources and verbs

```
GET     /users          → list users
GET     /users/42       → get user 42
POST    /users          → create a new user
PUT     /users/42       → replace user 42 entirely
PATCH   /users/42       → update some fields of user 42
DELETE  /users/42       → delete user 42

GET     /users/42/orders   → orders belonging to user 42 (nested resource)
```

Rule of thumb: **URLs are nouns, methods are verbs.** Write `POST /orders`, not `POST /createOrder`.

| Method | Purpose | Safe (no changes)? | Idempotent? |
|---|---|---|---|
| `GET` | Read | ✅ | ✅ |
| `POST` | Create / trigger an action | ❌ | ❌ |
| `PUT` | Replace | ❌ | ✅ |
| `PATCH` | Partial update | ❌ | Not guaranteed |
| `DELETE` | Remove | ❌ | ✅ |

**Idempotent** = doing it twice has the same effect as doing it once. This matters for retries: retrying a `DELETE` is harmless; blindly retrying a `POST /payments` might charge someone twice. (The usual fix is an **idempotency key** header.)

### Status codes

| Range | Meaning | Common ones |
|---|---|---|
| **2xx** | Success | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | Redirect | `301 Moved Permanently`, `304 Not Modified` |
| **4xx** | Client's fault | `400 Bad Request`, `401 Unauthorized` (not logged in), `403 Forbidden` (logged in, not allowed), `404 Not Found`, `409 Conflict`, `429 Too Many Requests` |
| **5xx** | Server's fault | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

### Statelessness

Each REST request must carry **everything needed to handle it** (e.g. the auth token). The server doesn't remember anything between requests. This is what lets you put any request on any server behind a load balancer. (More in `stateless-vs-stateful.md`.)

### Versioning

APIs change, but old clients (especially mobile apps) keep running. So you version:

| Style | Example |
|---|---|
| **URL path** (most common) | `/v1/users`, `/v2/users` |
| **Header** | `Accept: application/vnd.myapp.v2+json` |
| **Query param** | `/users?version=2` |

Adding a new field is usually safe. Removing or renaming one is a **breaking change** and needs a new version.

### Pagination

Never return 10 million rows in one response.

| Type | Example | Pros | Cons |
|---|---|---|---|
| **Offset** | `GET /posts?limit=20&offset=40` | Simple, can jump to page 5 | Slow on large offsets; items shift if data changes between pages |
| **Cursor** | `GET /posts?limit=20&after=abc123` | Fast and stable at any depth | Can't jump to an arbitrary page |

Cursor-based is what feeds (Twitter, Instagram) use. The cursor is usually an encoded ID or timestamp of the last item seen.

### REST example

```http
GET /v1/users/42
```

```json
{
  "id": 42,
  "name": "Asha",
  "email": "asha@example.com",
  "address": { "...": "..." },
  "createdAt": "2024-01-10T09:00:00Z"
}
```

**Weak spots:** you get the whole user even if you only needed `name` (over-fetching). And if you also want their last 3 orders and each order's product, that's several more calls (under-fetching).

---

## GraphQL

**GraphQL** has **one endpoint** (usually `POST /graphql`). The client sends a **query** shaped like the data it wants, and the response comes back in exactly that shape.

### Example

```graphql
query {
  user(id: 42) {
    name
    orders(last: 3) {
      total
      product { title }
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "Asha",
      "orders": [
        { "total": 59.99, "product": { "title": "Shoes" } },
        { "total": 12.00, "product": { "title": "Socks" } }
      ]
    }
  }
}
```

One round trip. No extra fields. With REST this would be ~3+ requests.

### Key ideas

| Concept | Meaning |
|---|---|
| **Schema** | A typed description of all data the API offers. The contract. |
| **Query** | Read data |
| **Mutation** | Change data (create/update/delete) |
| **Subscription** | Get pushed updates in real time (usually over WebSockets) |
| **Resolver** | The server function that fetches one field's data |

### The N+1 problem

Resolvers run per field. A naive server handling "10 users and each one's orders" does:

```
1 query   → SELECT * FROM users LIMIT 10
10 queries→ SELECT * FROM orders WHERE user_id = 1
            SELECT * FROM orders WHERE user_id = 2
            ... (one per user)
= 1 + N queries  ❌
```

**Fix: batching (the DataLoader pattern).** Collect all the user IDs in that step, then run one query: `SELECT * FROM orders WHERE user_id IN (1,2,...,10)`. Now it's 2 queries total ✅.

(N+1 can happen with REST and ORMs too. GraphQL just makes it very easy to trigger.)

### GraphQL trade-offs

- ✅ Client gets exactly what it needs. Great for mobile and for varied screens.
- ✅ Strongly typed schema, great tooling, easy to evolve without versions (add fields, deprecate old ones).
- ❌ **HTTP caching is harder.** Everything is a `POST` to one URL, so CDNs can't cache by URL easily. (Persisted queries help.)
- ❌ **Expensive queries.** A client can ask for deeply nested data. You need depth limits, complexity limits, and rate limiting.
- ❌ Errors often return `200 OK` with an `errors` field, which monitoring must understand.
- ❌ More server-side complexity.

---

## gRPC

**gRPC** (from Google) is **RPC = Remote Procedure Call**: you call a function on another machine as if it were a local function.

### Protobuf: the contract

You define services and messages in a `.proto` file:

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc StreamOrders (GetUserRequest) returns (stream Order);
}

message GetUserRequest { int32 id = 1; }

message User {
  int32  id   = 1;
  string name = 2;
}
```

A code generator turns this into client and server code in Go, Java, Python, etc. Calling it then looks like:

```go
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

**Protocol Buffers (protobuf)** encode data as compact **binary**, not text JSON. Smaller payloads and faster to parse. The numbers (`= 1`, `= 2`) are field tags, which is how it stays backward-compatible when you add fields.

### Built on HTTP/2

- **Multiplexing:** many calls over one connection at the same time.
- **Header compression:** less overhead per call.
- **Streaming:** built in.

### Four call types

| Type | Shape | Example |
|---|---|---|
| **Unary** | 1 request → 1 response | Get user |
| **Server streaming** | 1 request → many responses | Live stock prices |
| **Client streaming** | many requests → 1 response | Upload chunks of a file |
| **Bidirectional** | many ↔ many | Chat, real-time sync |

### Why it's mostly internal

```
                   PUBLIC (REST/GraphQL, JSON)     INTERNAL (gRPC, protobuf)
┌─────────┐            ┌─────────────┐        ┌───────────────┐
│ Browser │ ─────────► │ API Gateway │ ─────► │ User service  │
│ / app   │   JSON     └─────────────┘  gRPC  └───────┬───────┘
└─────────┘                   │                       │ gRPC
                              │ gRPC          ┌───────▼───────┐
                              └─────────────► │ Order service │
                                              └───────────────┘
```

- ❌ Browsers can't speak raw gRPC (they don't expose HTTP/2 framing control). You need **gRPC-Web** plus a proxy.
- ❌ Binary isn't human-readable. You can't just `curl` it and read the result.
- ✅ Between your own services, you control both sides, so the strict contract and speed are pure win.

---

## Side-by-side

| | REST | GraphQL | gRPC |
|---|---|---|---|
| **Endpoints** | Many (one per resource) | One | Service methods |
| **Data format** | JSON (usually) | JSON | Protobuf (binary) |
| **Transport** | HTTP/1.1 or 2 | HTTP (usually POST) | HTTP/2 |
| **Contract / schema** | Optional (OpenAPI) | Required (GraphQL schema) | Required (`.proto`) |
| **Over/under-fetching** | Common | Solved by design | Fixed by method design |
| **HTTP caching** | ✅ Easy (GET + URL) | ❌ Harder | ❌ Not really |
| **Streaming** | ❌ Not built in (use SSE/WebSockets) | Subscriptions | ✅ Built in, 4 modes |
| **Browser support** | ✅ Native | ✅ Native | ❌ Needs gRPC-Web |
| **Performance** | Good | Good (can hit N+1) | Best |
| **Human-readable** | ✅ | ✅ | ❌ |
| **Learning curve** | Low | Medium | Medium |

---

## When to pick which

| Situation | Pick |
|---|---|
| Public API for third-party developers | **REST**: everyone knows it, easy to cache and debug |
| Simple CRUD app | **REST** |
| Many different clients/screens needing different shapes of data (mobile + web + TV) | **GraphQL** |
| Frontend teams want to move fast without waiting for new backend endpoints | **GraphQL** |
| Microservices talking to each other, high volume, low latency | **gRPC** |
| Real-time streaming between services | **gRPC** |
| Polyglot backend (Go + Java + Python) needing strict contracts | **gRPC** |

Very common real combo: **REST or GraphQL at the edge (for clients), gRPC inside (between services).**

---

## Where this shows up in HLD

- In most interviews, after requirements you **define the API**: list 3-6 endpoints like `POST /tweets`, `GET /feed?cursor=...`. REST is the default unless you have a reason.
- **Pagination** comes up in any feed, search, or list design. Say "cursor-based" and explain why.
- **Idempotency** comes up in payments, orders, and anything retried: mention idempotency keys.
- **Microservice designs** often say "services talk over gRPC" for internal calls, and put an **API Gateway** in front that speaks REST/GraphQL to clients.
- GraphQL comes up with "mobile clients on slow networks" or "many different frontends". Be ready to mention N+1 and query cost limits.

---

## Key takeaways

- **REST** = resources + HTTP verbs + status codes. Stateless, cacheable, universal. The safe default.
- **GraphQL** = one endpoint, client picks exact fields. Fixes over/under-fetching; watch for N+1 and costly queries.
- **gRPC** = typed function calls over HTTP/2 with binary protobuf. Fast, streams, ideal for internal service-to-service.
- Use **cursor pagination** for large or changing lists, and **version** your public APIs.
- Know which methods are **idempotent**; it decides what's safe to retry.
- Real systems mix them: REST/GraphQL outside, gRPC inside.
