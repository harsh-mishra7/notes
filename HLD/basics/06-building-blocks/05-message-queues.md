# Message Queues

## Brief

**A message queue lets one part of a system hand off work to another part without waiting for it to finish.**

The sender (**producer**) drops a message into the queue and moves on. The receiver (**consumer**) picks it up whenever it's ready. The queue in the middle (run by a **broker**) holds messages safely until they're processed.

This turns "do it now and make the user wait" into "note it down, reply instantly, do it in the background."

---

## The problem it solves: synchronous chains are fragile

A user signs up. Your API needs to: save the user, send a welcome email, resize their profile photo, and notify the analytics system.

```
SYNCHRONOUS (everything in the request)

User ──► API ──► save to DB        20 ms
             ──► send email       800 ms   (email provider is slow)
             ──► resize photo     1.5 s
             ──► analytics         200 ms
         ◄── response after ~2.5 s ❌   and if email service is down → signup FAILS ❌


ASYNCHRONOUS (with a queue)

User ──► API ──► save to DB        20 ms
             ──► put "user_signed_up" in queue   2 ms
         ◄── response after ~25 ms ✅

                  [ queue ] ──► email worker     (whenever it's ready)
                            ──► photo worker
                            ──► analytics worker
```

The user gets a fast response. If the email service is down for 10 minutes, messages just wait in the queue and get sent when it's back.

---

## The analogy that makes it click

**A message queue is the order ticket rail in a restaurant kitchen.**

The waiter (producer) takes your order, clips the ticket to the rail, and goes back to serving other tables. They don't stand in the kitchen waiting for your food. The cooks (consumers) grab tickets from the rail one by one, at their own pace.

- Dinner rush? Tickets pile up on the rail — nothing is lost, the cooks just work through them. (**buffering**)
- Too many tickets? Add another cook. (**scaling consumers**)
- A cook burns a dish? The ticket goes back up to be remade. (**retries**)
- The waiter doesn't need to know which cook makes which dish. (**decoupling**)

---

## The vocabulary

| Term | Meaning |
|---|---|
| **Message** | A small piece of data describing work or an event (`{"event": "order_placed", "id": 123}`) |
| **Producer** | The service that sends messages |
| **Consumer** | The service that reads and processes messages |
| **Broker** | The server/system that stores and delivers messages (RabbitMQ, Kafka, SQS) |
| **Queue / topic** | A named channel messages go into |
| **Ack** (acknowledgement) | Consumer telling the broker "I'm done with this one, you can delete it" |
| **DLQ** (Dead-Letter Queue) | A side queue for messages that keep failing |

---

## Sync vs async communication

| | **Synchronous** (HTTP/gRPC call) | **Asynchronous** (message queue) |
|---|---|---|
| Caller waits? | Yes, until the reply comes back | No, fire and move on |
| If receiver is down | Caller's request fails | Message waits in the queue |
| Coupling | Tight — caller must know the receiver | Loose — producer only knows the queue |
| Latency for user | Sum of all calls | Just the time to enqueue |
| Getting a result back | Easy, it's the response | Harder — needs callbacks, polling, or another queue |
| Best for | Reads, anything the user needs an answer to *now* | Emails, notifications, video processing, analytics, anything that can happen "soon" |

---

## Two models: point-to-point vs pub-sub

### Point-to-point (work queue)

Each message is processed by **exactly one** consumer. Multiple consumers share the work.

```
                         ┌──► Worker 1   (gets msg 1, 4)
Producer ──► [ 1 2 3 4 ] ├──► Worker 2   (gets msg 2, 5)
                         └──► Worker 3   (gets msg 3, 6)

Each message is handled once. Add workers to go faster.
```

Use for: **jobs / tasks** — "resize this image", "send this email".

### Publish-Subscribe (pub-sub)

Each message is delivered to **every** subscriber. Each subscriber gets its own copy.

```
                                  ┌──► Email service      (gets a copy)
Producer ──► topic "order_placed" ├──► Inventory service  (gets a copy)
                                  └──► Analytics service  (gets a copy)

One event, many independent reactions.
```

Use for: **events** — "an order was placed; whoever cares, react to it." Adding a new subscriber needs no change to the producer.

| | **Point-to-point** | **Pub-sub** |
|---|---|---|
| Who gets a message | One consumer | Every subscriber |
| Think of it as | A to-do list shared by workers | A newsletter / broadcast |
| Example use | Background jobs | Event-driven microservices |

---

## Why use a message queue

| Benefit | What it means |
|---|---|
| **Decoupling** | Producer doesn't know or care who consumes. Services can be changed, deployed, or scaled independently. |
| **Buffering spikes (load leveling)** | 10,000 orders arrive in one second; workers process 500/s. The queue holds the backlog instead of the system crashing. |
| **Resilience** | If a consumer is down, messages wait. Nothing is lost. |
| **Retries** | A failed message can be retried automatically instead of the whole request failing. |
| **Scaling** | Queue getting long? Add more consumers. |
| **Faster responses** | Slow work moves out of the user's request path. |

```
Traffic spike (load leveling):

incoming:   ▁▁▁█████▁▁▁▁▁▁          (spiky)
queue:      ▁▁▁▃▅▇█▇▅▃▁▁▁▁          (absorbs the spike)
processed:  ▁▁▁▄▄▄▄▄▄▄▄▁▁▁          (steady, within capacity)
```

---

## Delivery guarantees

Networks fail and consumers crash mid-work. So how many times does a message actually get processed?

| Guarantee | Meaning | Risk | How |
|---|---|---|---|
| **At-most-once** | Delivered 0 or 1 times | Messages can be **lost** | Ack before processing (or never retry) |
| **At-least-once** | Delivered 1 or more times | **Duplicates** possible | Ack only after processing; retry if no ack |
| **Exactly-once** | Processed exactly 1 time | Hard, costly, limited scope | Transactions / dedup inside the system (e.g. Kafka transactions) |

```
Why at-least-once causes duplicates:

Consumer: gets msg ──► charges card ✅ ──► crashes before sending ack ❌
Broker:   no ack received ──► redelivers msg ──► card charged AGAIN ❌
```

**At-least-once is the practical default.** Make consumers **idempotent** — processing the same message twice has the same effect as once. Common trick: give each message a unique ID and skip IDs you've already processed.

---

## Ordering

Do messages come out in the order they went in?

- **Simple queues often don't guarantee strict order**, especially with many consumers in parallel (worker 2 might finish msg 2 before worker 1 finishes msg 1). Retries also reorder things.
- **Kafka** guarantees order **within a partition**. Send all messages for the same key (e.g. same `user_id`) to the same partition, and they stay in order for that user.
- **SQS FIFO** queues guarantee order within a **message group**.

**Trade-off:** strict global ordering limits parallelism. Usually you only need ordering *per entity* (per user, per order), not globally.

---

## Dead-letter queues (DLQ)

Some messages will fail every time — bad data, a bug, a missing record. Retrying forever blocks the queue and wastes resources (a "poison message").

```
Main queue ──► Consumer ── fails ──► retry (1) ── fails ──► retry (2) ── fails ──► retry (3)
                                                                                     │
                                                                           still failing
                                                                                     ▼
                                                                        [ Dead-Letter Queue ]
                                                                     (inspect, fix, replay later)
```

After N failed attempts, the message moves to a DLQ. The main queue keeps flowing, and engineers can look at the DLQ, fix the bug, and replay the messages. Set up alerts on DLQ size.

---

## Kafka vs RabbitMQ vs SQS

| | **Apache Kafka** | **RabbitMQ** | **Amazon SQS** |
|---|---|---|---|
| What it is | Distributed, append-only **log** | Traditional **message broker** | Fully **managed queue** service (AWS) |
| After consuming | Messages **stay** (kept for a retention period) — can be re-read | Deleted once acked | Deleted once processed |
| Model | Pub-sub via topics + consumer groups | Queues, flexible routing (exchanges), pub-sub | Point-to-point (use SNS for pub-sub fan-out) |
| Throughput | Very high (millions msgs/sec) | High (tens of thousands msgs/sec per node, typical) | Scales automatically, high |
| Ordering | Per partition | Per queue (with one consumer) | Standard: best-effort; FIFO queues: per group |
| Replay old messages | ✅ Yes | ❌ No (not by default) | ❌ No |
| Operations | Complex to run yourself | Moderate | None — AWS runs it |
| Best for | Event streaming, logs, analytics pipelines, event sourcing | Task queues, complex routing, request/reply | Simple, reliable background jobs on AWS |

**Quick pick:**
- Need to **stream huge volumes of events** that multiple systems read (and maybe re-read)? → **Kafka**
- Need a **classic job queue** with flexible routing? → **RabbitMQ**
- On AWS and want **zero ops**? → **SQS** (+ SNS for fan-out)

Others you'll hear: Google Pub/Sub, Azure Service Bus, Redis Streams, NATS, Apache Pulsar.
