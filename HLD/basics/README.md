# HLD Basics

The foundations to know before going deep into High Level Design.
Read the folders in order: each one builds on the one before it.

---

## 1. [Networking Fundamentals](01-networking-fundamentals/)

| Note | What it covers |
|---|---|
| [How a request travels](01-networking-fundamentals/how-a-request-travels.md) | URL → DNS → TCP → TLS → HTTP → response |
| [IP, ports, TCP vs UDP](01-networking-fundamentals/ip-ports-tcp-udp.md) | Addressing, and reliability vs speed |
| [HTTP / HTTPS](01-networking-fundamentals/http-https.md) | Methods, status codes, headers, HTTP/1.1 vs 2 vs 3 |
| [DNS](01-networking-fundamentals/dns.md) | Name resolution, record types, TTL, caching |
| [Real-time communication](01-networking-fundamentals/realtime-communication.md) | Polling, long polling, SSE, WebSockets |

---

## 2. [Client–Server and APIs](02-client-server-and-apis/)

| Note | What it covers |
|---|---|
| [Client–server model](02-client-server-and-apis/client-server-model.md) | Request/response, tiers |
| [API styles](02-client-server-and-apis/api-styles.md) | REST vs GraphQL vs gRPC |
| [Stateless vs stateful](02-client-server-and-apis/stateless-vs-stateful.md) | Why stateless scales |
| [Authentication and authorization](02-client-server-and-apis/authentication-and-authorization.md) | Sessions, cookies, JWT, OAuth |

---

## 3. [Core Scalability Concepts](03-core-scalability-concepts/)

| Note | What it covers |
|---|---|
| [Vertical vs horizontal scaling](03-core-scalability-concepts/vertical-vs-horizontal-scaling.md) | Scale up vs scale out |
| [Latency vs throughput](03-core-scalability-concepts/latency-vs-throughput.md) | Percentiles, latency numbers |
| [Availability vs reliability](03-core-scalability-concepts/availability-vs-reliability.md) | The nines, SLA/SLO/SLI |
| [Single point of failure](03-core-scalability-concepts/single-point-of-failure.md) | Redundancy and failover |

---

## 4. [Databases](04-databases/)

| Note | What it covers |
|---|---|
| [SQL vs NoSQL](04-databases/sql-vs-nosql.md) | Relational vs document / key-value / wide-column / graph |
| [Indexing](04-databases/indexing.md) | B-trees, composite indexes, write cost |
| [ACID and transactions](04-databases/acid-and-transactions.md) | Isolation levels and anomalies |
| [Replication](04-databases/replication.md) | Leader–follower, read replicas, lag |
| [Sharding and partitioning](04-databases/sharding-and-partitioning.md) | Shard keys, hot spots |
| [Normalization vs denormalization](04-databases/normalization-vs-denormalization.md) | Consistency vs read speed |

---

## 5. [Distributed Systems Theory](05-distributed-systems-theory/)

| Note | What it covers |
|---|---|
| [CAP theorem](05-distributed-systems-theory/cap-theorem.md) | CP vs AP, PACELC |
| [Consistency models](05-distributed-systems-theory/consistency-models.md) | Strong vs eventual, quorums |
| [Consistent hashing](05-distributed-systems-theory/consistent-hashing.md) | The hash ring, virtual nodes |
| [Idempotency, retries, timeouts](05-distributed-systems-theory/idempotency-retries-timeouts.md) | Surviving an unreliable network |

---

## 6. [Building Blocks](06-building-blocks/)

| Note | What it covers |
|---|---|
| [Load balancer](06-building-blocks/load-balancer.md) | L4 vs L7, algorithms, health checks |
| [Reverse proxy and API gateway](06-building-blocks/reverse-proxy-and-api-gateway.md) | How they relate and overlap |
| [Caching](06-building-blocks/caching.md) | Cache-aside, write-through, eviction |
| [CDN](06-building-blocks/cdn.md) | Edge servers, hit/miss, TTL |
| [Message queues](06-building-blocks/message-queues.md) | Async processing, pub-sub, Kafka vs RabbitMQ |
| [Object storage](06-building-blocks/object-storage.md) | S3-style blobs, pre-signed URLs |
| [Rate limiter](06-building-blocks/rate-limiter.md) | Token bucket, sliding window |

---

## The mindset

HLD has no single right answer. For every component you add, be able to say
**why you chose it and what you gave up.**
