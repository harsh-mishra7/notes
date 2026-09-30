# Real-time Communication

## Brief

Normal HTTP is **client asks, server answers**. The server can't speak first. But chat messages, notifications, and live scores need the server to push updates *as they happen*.

Four common ways to get there:

| Technique | One-line idea |
|---|---|
| **Short polling** | Client asks "anything new?" every few seconds |
| **Long polling** | Client asks, server *holds* the request open until something new happens |
| **Server-Sent Events (SSE)** | One long-lived HTTP response; server streams events down it, one-way |
| **WebSockets** | A persistent, two-way connection; both sides send whenever they like |

Rough rule: **SSE** for server → client updates, **WebSockets** for true two-way interaction, **polling** when updates are rare or you need the simplest thing that works.

---

## The analogy that makes it click

**You're waiting for a package.**

- **Short polling:** you call the courier every 5 minutes. "Is it here?" "No." "Is it here?" "No." Annoying and wasteful.
- **Long polling:** you call once and the courier says "hold the line, I'll tell you the moment it arrives." When it does, they tell you, you hang up, and call again.
- **SSE:** you subscribe to SMS updates. The courier texts you each time something changes. You can't text back on that channel.
- **WebSocket:** you and the courier are on an open phone call. Either of you can talk at any moment.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Pull** | Client asks for data |
| **Push** | Server sends data without being asked |
| **Persistent connection** | A connection that stays open for a long time instead of closing after one response |
| **Full-duplex** | Both sides can send at the same time |
| **Stateful server** | Server must remember which client is on which connection |

---

## 1. Short polling

The client repeats a normal request on a timer.

```
Client                               Server
  │ ── GET /messages?since=10 ──────►  │
  │ ◄────────────────── [] (nothing) ─ │
  │        ...wait 5s...               │
  │ ── GET /messages?since=10 ──────►  │
  │ ◄────────────────── [] (nothing) ─ │
  │        ...wait 5s...               │
  │ ── GET /messages?since=10 ──────►  │
  │ ◄──────────── [msg 11] ──────────  │   finally something
```

| Pros | Cons |
|---|---|
| Dead simple — just normal HTTP | Most requests return nothing (wasted work) |
| Works everywhere, stateless, easy to scale and cache | Delay up to the polling interval |
| | Shorter interval = fresher data but way more load |

**Load math:** 1 million clients polling every 5 s = **200,000 requests/second**, mostly empty.

---

## 2. Long polling

The client asks; the server **doesn't answer until it has something** (or a timeout, e.g. 30 s). Then the client immediately asks again.

```
Client                               Server
  │ ── GET /messages?since=10 ──────►  │
  │                                    │   (holding... nothing yet)
  │                                    │   (holding...)
  │ ◄──────────── [msg 11] ──────────  │   new message → respond now
  │ ── GET /messages?since=11 ──────►  │   reconnect immediately
  │                                    │   (holding...)
  │ ◄──────────── [] (timeout 30s) ─── │   nothing came → empty response
  │ ── GET /messages?since=11 ──────►  │   reconnect
```

| Pros | Cons |
|---|---|
| Near-instant delivery | Server holds many open requests (needs async/event-driven servers) |
| Far fewer empty responses than short polling | Still one HTTP request per message batch (headers overhead) |
| Plain HTTP, works through most proxies/firewalls | Messages can slip through in the gap between responses unless you track a cursor (`since=`) |
| Good fallback when WebSockets are blocked | Timeouts at proxies/LBs must be tuned |

---

## 3. Server-Sent Events (SSE)

The client opens **one HTTP request**, and the server **never finishes the response**. It keeps writing events into it.

```
Client                                  Server
  │ ── GET /stream                     │
  │    Accept: text/event-stream ────► │
  │                                    │
  │ ◄── HTTP 200                       │
  │     Content-Type: text/event-stream│
  │ ◄── data: {"score":"1-0"}          │   event
  │ ◄── data: {"score":"2-0"}          │   event
  │ ◄── data: {"score":"2-1"}          │   event
  │           ... stays open ...       │
```

What the stream looks like on the wire:

```
id: 42
event: score
data: {"home": 2, "away": 1}

```

In the browser:

```js
const es = new EventSource("/stream");
es.onmessage = (e) => console.log(JSON.parse(e.data));
```

| Pros | Cons |
|---|---|
| Plain HTTP — works with existing auth, LBs, HTTP/2 | **One-way only** (server → client); client sends via normal requests |
| **Auto-reconnect built in**, resumes via `Last-Event-ID` | Text only (binary must be encoded) |
| Very simple on both client and server | Over HTTP/1.1, browsers allow ~6 connections per domain (not an issue with HTTP/2) |
| Efficient: no per-message request overhead | Still one open connection per client |

---

## 4. WebSockets

Starts as an HTTP request that asks to **upgrade** the connection. After that, it's no longer HTTP — it's a raw, two-way message channel over the same TCP connection.

```
Client                                   Server
  │ ── GET /chat HTTP/1.1               │
  │    Upgrade: websocket               │
  │    Connection: Upgrade ───────────► │
  │                                     │
  │ ◄── 101 Switching Protocols ─────── │
  │                                     │
  │ ═════════ WebSocket open ═════════  │
  │ ── "hi" ──────────────────────────► │
  │ ◄───────────────────── "hello!" ─── │
  │ ◄──────────── "user X is typing" ── │
  │ ── "how are you?" ────────────────► │
  │         ... both sides, anytime ... │
```

In the browser:

```js
const ws = new WebSocket("wss://chat.example.com/chat");
ws.onmessage = (e) => console.log(e.data);
ws.send("hi");
```

| Pros | Cons |
|---|---|
| **Full-duplex**: both sides send anytime | **Stateful**: each client pinned to one server connection — harder to scale |
| Lowest latency and overhead per message (tiny frame headers) | No built-in reconnect or message replay — you build it |
| Supports text and binary | Some corporate proxies/firewalls interfere (use `wss://` to help) |
| | Load balancers need long idle timeouts and WebSocket support |

---

## Comparison table

| | Short polling | Long polling | SSE | WebSockets |
|---|---|---|---|---|
| Direction | Client pulls | Client pulls (server holds) | Server → client | Both ways |
| Latency | Up to poll interval | Near real-time | Real-time | Real-time |
| Connection | New request each time | Request held, then repeated | One long HTTP response | One persistent TCP connection |
| Overhead per update | High (full request, often empty) | Medium (full request per update) | Low | Lowest |
| Protocol | HTTP | HTTP | HTTP | WebSocket (`ws://` / `wss://`) |
| Auto-reconnect | N/A | Manual | ✅ Built in | ❌ Build it yourself |
| Binary data | ✅ | ✅ | ❌ (text) | ✅ |
| Server state | Stateless | Holds open requests | Holds open connections | Holds open connections |
| Complexity | Lowest | Low | Low | Highest |

---

## When to use which

| Use case | Best fit | Why |
|---|---|---|
| **Chat app** (WhatsApp Web, Slack) | WebSockets | Two-way, frequent messages, typing indicators, presence |
| **Multiplayer game / collaborative editing** (Figma, Google Docs) | WebSockets | Constant two-way updates, low latency |
| **Notifications** ("you have a new follower") | SSE (or long polling) | One-way, infrequent; SSE is simple and reconnects on its own |
| **Live sports scores** | SSE | Server broadcasts updates; users only watch |
| **Stock tickers** | SSE or WebSockets | SSE for view-only price streams; WebSockets if users also place orders/subscribe dynamically on the same channel |
| **AI chat responses streaming token by token** | SSE | One-way stream per request; plain HTTP |
| **Order / delivery status** (updates every few minutes) | Short or long polling | Updates are rare; simplest is fine |
| **Dashboard refreshing every 30-60 s** | Short polling | Freshness needs are low; caching friendly |

**Decision shortcut:**

```
Does the client need to send frequent messages back on the same channel?
   ├── Yes ──► WebSockets
   └── No  ──► Do updates need to be near-instant?
                  ├── Yes ──► SSE   (long polling as a fallback)
                  └── No  ──► Short polling
```

---

## Scaling persistent connections (the HLD part)

SSE and WebSockets mean **every online user holds an open connection**. With 10 million users online, that's 10 million connections across your fleet.

Problem: user A is connected to server 1, user B to server 3. A sends B a message — how does server 1 reach B?

```
   User A                                          User B
     │                                               ▲
     ▼                                               │
┌──────────┐     publish "msg for B"     ┌──────────┐
│ WS server│ ──────────►┌────────┐──────►│ WS server│
│    1     │            │ Pub/Sub│       │    3     │
└──────────┘            │ (Redis,│       └──────────┘
                        │ Kafka) │
┌──────────┐            └────────┘
│ WS server│   (servers subscribe to channels for the users they hold)
│    2     │
└──────────┘
```

Common building blocks:

- **Pub/Sub layer** (Redis Pub/Sub, Kafka, NATS) to route messages between connection servers.
- **Connection registry** (e.g. Redis: `userB → server 3`) to know where each user lives.
- **Sticky sessions / consistent routing** at the load balancer for long-lived connections.
- **Heartbeats (ping/pong)** to detect dead connections and keep idle proxies from cutting them.
- **Reconnect + resume** logic: clients reconnect with backoff and ask for anything they missed (via a last-seen message ID).
