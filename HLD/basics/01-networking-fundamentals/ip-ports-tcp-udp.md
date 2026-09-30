# IP, Ports, TCP and UDP

## TL;DR

- **IP address** = *which machine* on the network.
- **Port** = *which program* on that machine.
- **TCP** = reliable delivery. Everything arrives, in order, or you find out it didn't. Slower to start.
- **UDP** = fire-and-forget. Fast and light, but packets can be lost or arrive out of order.

Web pages, APIs, and databases use **TCP**. Video calls, online games, and DNS lean on **UDP**, because for them *late* data is as bad as *lost* data.

---

## The analogy that makes it click

**The internet is a giant postal system.**

- **IP address** = the building's street address.
- **Port** = the apartment number inside the building. One building (machine) has many residents (programs).
- **TCP** = registered mail with tracking. Every letter is numbered, signed for, and resent if lost. The receiver reads them in order.
- **UDP** = dropping postcards in a mailbox. Cheap and quick, but some may never arrive, and nobody tells you.

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Packet** | A small chunk of data sent across the network (typically ~1,500 bytes max on Ethernet) |
| **IP** | Internet Protocol. Gets packets from one machine to another, *best effort*, no guarantees. |
| **Protocol** | A set of rules both sides agree on |
| **Socket** | One end of a connection: `IP + port` (e.g. `10.0.0.5:8080`) |
| **Latency** | How long one message takes to arrive |
| **Throughput** | How much data per second you can push |

---

## The layers (just enough)

Networking is built in layers. Each layer only cares about its own job:

```
┌─────────────────────────────────────────────┐
│ Application   HTTP, DNS, WebSocket, SMTP    │  "What are we saying?"
├─────────────────────────────────────────────┤
│ Transport     TCP, UDP  (+ ports)           │  "Which program? Reliable or not?"
├─────────────────────────────────────────────┤
│ Network       IP  (addresses, routing)      │  "Which machine?"
├─────────────────────────────────────────────┤
│ Link          Ethernet, Wi-Fi               │  "Next hop on this wire"
└─────────────────────────────────────────────┘
```

HTTP rides on TCP, which rides on IP. That's the stack for most of the web.

---

## IP addresses

### IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Example | `142.250.183.14` | `2404:6800:4009:82b::200e` |
| Size | 32 bits | 128 bits |
| Total addresses | ~4.3 billion | ~3.4 × 10³⁸ (effectively unlimited) |
| Status | Ran out years ago; still dominant | Growing adoption, runs alongside IPv4 |

IPv4 ran out because there are more devices than addresses. The workarounds are **NAT** (below) and moving to IPv6.

### Public vs private

| | Public IP | Private IP |
|---|---|---|
| Reachable from the internet? | ✅ Yes | ❌ No |
| Unique worldwide? | Yes | No — reused in every home/office |
| Ranges (IPv4) | Everything else | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| Example | Your router's outside address | Your laptop: `192.168.1.23` |

Special one: `127.0.0.1` (`localhost`) always means "this machine".

### NAT: how many devices share one public IP

```
  Home network (private)                       Internet (public)

  Laptop  192.168.1.23 ─┐
  Phone   192.168.1.24 ─┼──► Router ──────────► 49.36.12.7 ──► google.com
  TV      192.168.1.25 ─┘    (NAT)
                             rewrites private → public,
                             remembers who asked for what
```

The router swaps the private source address for its public one, and uses port numbers to route replies back to the right device.

---

## Ports

A machine runs many programs. The **port** (a number from 0 to 65535) says which one a packet is for.

```
Server 10.0.0.5
 ├── :22    SSH
 ├── :443   Web server (HTTPS)
 ├── :5432  PostgreSQL
 └── :6379  Redis
```

| Port | Service |
|---|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |

- **0-1023**: well-known ports (standard services)
- **49152-65535**: ephemeral ports — the OS picks one randomly for *your side* of an outgoing connection

A connection is uniquely identified by **source IP + source port + destination IP + destination port + protocol**. That's how one server handles thousands of clients on port 443 at once.

---

## TCP: the reliable one

TCP turns IP's "best effort" into a **reliable, ordered byte stream**.

### 1. The 3-way handshake (setup)

```
Client                                 Server
   │ ── SYN (seq=100) ───────────────►  │   "I want to connect. My numbers start at 100."
   │ ◄── SYN-ACK (seq=300, ack=101) ──  │   "OK. Mine start at 300. Got your 100."
   │ ── ACK (ack=301) ───────────────►  │   "Got your 300."
   │                                    │
   │ ═══════ data can now flow ═══════  │
```

Costs **1 RTT** before sending data. Both sides now know the other is alive and have agreed on starting sequence numbers.

### 2. Reliability (acknowledgements + retransmission)

Every segment the receiver gets is **ACKed**. If the sender doesn't get an ACK in time, it resends.

```
Sender                 Receiver
  │ ── seg 1 ──────────►  │
  │ ◄───────── ACK 1 ──── │
  │ ── seg 2 ────X        │   (lost)
  │    ...timeout...      │
  │ ── seg 2 ──────────►  │   (resent)
  │ ◄───────── ACK 2 ──── │
```

### 3. Ordering

Packets can take different routes and arrive shuffled. TCP uses **sequence numbers** to put them back in order before handing data to the app.

```
Arrive:  [3] [1] [2]   ──TCP reorders──►   App sees: [1] [2] [3]
```

Downside — **head-of-line blocking**: if packet 1 is lost, packets 2 and 3 wait in a buffer until 1 is resent, even though they're already here.

### 4. Flow control (don't drown the receiver)

The receiver advertises a **window**: "I have room for 64 KB more." The sender never sends more than that. Protects a slow *receiver*.

### 5. Congestion control (don't drown the network)

TCP starts slow (**slow start**), speeds up while things go well, and backs off sharply when it detects loss. Protects the shared *network*. This is why a fresh connection is slower than a warmed-up one.

### 6. Teardown

Closing uses `FIN` / `ACK` messages from both sides, so neither side loses data mid-close.

---

## UDP: the fast one

UDP adds almost nothing on top of IP: just ports and a checksum.

```
Client                    Server
  │ ── datagram ────────►   │    no handshake
  │ ── datagram ────X       │    lost? nobody resends
  │ ── datagram ────────►   │    may arrive out of order
```

- No handshake → first packet carries data immediately
- No ACKs, no retransmits, no ordering
- Tiny 8-byte header (TCP's is 20+ bytes)
- If the app needs reliability, **the app builds it** (that's exactly what QUIC does)

---

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | Yes (handshake) | No |
| Delivery guaranteed | ✅ | ❌ |
| Order guaranteed | ✅ | ❌ |
| Flow / congestion control | ✅ Built in | ❌ App's job |
| Speed to first byte | Slower (1 RTT setup) | Immediate |
| Header size | 20-60 bytes | 8 bytes |
| Data model | Continuous byte stream | Individual messages (datagrams) |
| Broadcast / multicast | ❌ | ✅ |

---

## When each is used

| Use case | Protocol | Why |
|---|---|---|
| Web pages, REST APIs | TCP | A missing byte breaks the page |
| Database connections | TCP | Queries must arrive complete and in order |
| File downloads, email | TCP | Correctness over speed |
| **Video / voice calls** | UDP | A late frame is useless; skip it and keep going |
| **Online games** | UDP | Only the *latest* position matters |
| **DNS lookups** | UDP (TCP for large responses) | One tiny question, one tiny answer — a handshake would double the cost |
| Live streaming (low-latency) | UDP | Real-time beats perfect |
| HTTP/3 (QUIC) | UDP | Builds its own smarter reliability on top |

> Note: Netflix/YouTube *on-demand* video mostly uses TCP (HTTP). Buffering a few seconds ahead hides retransmission delays. It's **real-time** media that prefers UDP.

**Rule of thumb:** if losing data is worse than waiting → TCP. If waiting is worse than losing a bit → UDP.

---

## Where this shows up in HLD

- **Protocol choice:** designing a video call app, a multiplayer game, or a metrics pipeline? Say *why* you'd pick UDP over TCP.
- **Connection cost:** TCP handshakes explain why **connection pooling** (to databases) and **keep-alive** (for HTTP) matter at scale.
- **Load balancers:** L4 load balancers route on IP + port (TCP/UDP level); L7 ones read HTTP. Knowing the layers makes this distinction clear.
- **Private networks:** in cloud designs, app servers and databases live on **private IPs** inside a VPC; only the load balancer has a public one.
- **Limits:** each connection needs a port and memory — relevant when designing systems with millions of open connections (chat, WebSockets).

---

## Key takeaways

- **IP** finds the machine, **port** finds the program; together they form a socket.
- Private IPs are reused everywhere; **NAT** lets them share a public IP.
- **TCP** = handshake + ACKs + ordering + flow/congestion control → reliable but slower to start.
- **UDP** = no guarantees, minimal overhead → ideal for real-time data where late is useless.
- Pick TCP when correctness matters most, UDP when freshness matters most.
