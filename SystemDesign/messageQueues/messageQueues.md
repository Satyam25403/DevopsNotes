# 📨 Message Queues & Event-Driven Architecture

> **Series:** System Design Notes  
> **Module:** 07 — Message Queues & Event-Driven Architecture  
> **Prerequisites:** `02_load_balancing.md`, `05_scalability.md`, Basic distributed systems concepts

---

## 📌 Table of Contents

1. [What is a Message Queue?](#1-what-is-a-message-queue)
2. [Core Terminology](#2-core-terminology)
3. [Synchronous vs Asynchronous Communication](#3-synchronous-vs-asynchronous-communication)
4. [Messaging Patterns](#4-messaging-patterns)
5. [Message Delivery Guarantees](#5-message-delivery-guarantees)
6. [Kafka — Deep Dive](#6-kafka--deep-dive)
7. [RabbitMQ — Deep Dive](#7-rabbitmq--deep-dive)
8. [AWS SQS + SNS](#8-aws-sqs--sns)
9. [Kafka vs RabbitMQ vs SQS](#9-kafka-vs-rabbitmq-vs-sqs)
10. [Dead Letter Queues (DLQ)](#10-dead-letter-queues-dlq)
11. [Idempotency](#11-idempotency)
12. [Event-Driven Architecture (EDA)](#12-event-driven-architecture-eda)
13. [Real-World Architectures](#13-real-world-architectures)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview Cheatsheet](#15-interview-cheatsheet)

---

## 1. What is a Message Queue?

> **Definition:** A message queue is a durable intermediary that enables services to communicate **asynchronously** by sending messages to a queue instead of calling each other directly. The producer and consumer are fully decoupled — they never need to be online simultaneously.

Think of it like a **post office**. You drop a letter (message) at the post office (queue). The recipient picks it up whenever they're ready. You don't wait at the post office — you go about your business.

```
WITHOUT MESSAGE QUEUE (tight coupling):
  Order Service ──HTTP──► Payment Service ──HTTP──► Inventory Service
                (must be up)               (must be up)
  If Payment is down → entire order fails

WITH MESSAGE QUEUE (loose coupling):
  Order Service ──► [ Queue ] ◄── Payment Service (reads when ready)
                    [ Queue ] ◄── Inventory Service (reads when ready)
                    [ Queue ] ◄── Notification Service (reads when ready)

  If Payment is down → messages accumulate in queue safely
  When Payment recovers → processes all pending messages ✅
```

**Why queues exist — the core problems they solve:**

| Problem | Queue Solution |
|---|---|
| **Tight coupling** | Producer doesn't know or care who consumers are |
| **Traffic spikes** | Queue acts as a buffer — absorbs burst, smooths load |
| **Speed mismatch** | Fast producer + slow consumer → queue bridges the gap |
| **Cascading failures** | Downstream failure doesn't propagate upstream |
| **Fanout** | One message → delivered to many independent consumers |
| **Retry on failure** | Failed processing → message stays in queue / goes to DLQ |

---

## 2. Core Terminology

| Term | Definition |
|---|---|
| **Producer** | Service that creates and sends messages to the queue/broker |
| **Consumer** | Service that reads and processes messages from the queue |
| **Broker** | The server managing the queue (RabbitMQ broker, Kafka broker) |
| **Queue** | Ordered buffer of messages; consumed messages are deleted (RabbitMQ/SQS) |
| **Topic** | Named channel for messages; messages are retained and replayable (Kafka) |
| **Message** | Unit of data — usually JSON, Avro, or Protobuf payload + metadata |
| **Offset** | Kafka-specific: integer position of a message in a partition |
| **Partition** | Kafka-specific: ordered sub-log of a topic for parallelism |
| **Consumer Group** | Group of consumers sharing consumption of a topic's partitions |
| **Acknowledgement (ACK)** | Consumer confirms message was processed → broker deletes it |
| **NACK** | Negative ack — consumer signals failure → message requeued |
| **Dead Letter Queue (DLQ)** | Separate queue for messages that repeatedly fail processing |
| **Idempotency** | Processing the same message twice produces the same result |
| **Backpressure** | Consumer signals producer to slow down when overwhelmed |
| **Consumer Lag** | Gap between latest message written and latest message processed |
| **Retention** | How long the broker keeps messages (Kafka: time or size limit) |
| **Log Compaction** | Kafka: keep only the latest value per key, discard old versions |

---

## 3. Synchronous vs Asynchronous Communication

```
SYNCHRONOUS (HTTP / gRPC):
  Client sends request → waits → Server processes → Client gets response

  Client ──request──► Server
         ◄──response──

  ✅ Simple, immediate feedback
  ❌ Tight coupling — if server is slow/down, client blocks/fails
  ❌ Hard to fan out to multiple services
  ❌ Traffic spikes → server overload

ASYNCHRONOUS (Message Queue):
  Producer sends message → returns immediately → Consumer processes independently

  Producer ──message──► [ Queue ] ──message──► Consumer
  (returns immediately)            (processes at its own pace)

  ✅ Decoupled — services don't know about each other
  ✅ Resilient — consumer failure doesn't affect producer
  ✅ Buffer absorbs traffic spikes
  ✅ Trivial fanout — add new consumer, zero producer changes
  ❌ No immediate response (eventual consistency)
  ❌ Harder to debug (no linear request trace)
  ❌ Extra infrastructure to manage
```

### When to Use Which

```
Use SYNCHRONOUS (HTTP/gRPC) when:
  ✅ Need an immediate response (user login, payment confirmation)
  ✅ Simple request-response flow
  ✅ Low latency critical (sub-100ms)

Use ASYNCHRONOUS (Queue) when:
  ✅ Processing can happen later (email, notifications, image resizing)
  ✅ Need to fan out to multiple services
  ✅ Consumer is significantly slower than producer
  ✅ Need to absorb traffic spikes (flash sale order processing)
  ✅ Need retry / durability guarantees
  ✅ Event streaming / audit logs
```

---

## 4. Messaging Patterns

### 4.1 Point-to-Point (Task Queue / Work Queue)

> One producer, one consumer processes each message. Multiple consumers **compete** for messages — each message goes to exactly one consumer.

```
Producer ──► [ Queue ] ──► Consumer 1 (gets message A)
                      ──► Consumer 2 (gets message B)
                      ──► Consumer 3 (gets message C)

Each message processed by EXACTLY ONE consumer.
Load is distributed across consumer pool.
```

```
Real examples:
  - Job queue: resize image tasks distributed to worker pool
  - Order processing: each order processed by one worker
  - Email sending: each send task handled by one mailer
```

**Used by:** RabbitMQ (default), SQS, Redis Queue

---

### 4.2 Publish-Subscribe (Pub/Sub / Fan-Out)

> One producer publishes to a **topic**. Every subscriber gets a copy of every message. Multiple independent consumers can each process all messages.

```
Producer ──► [ Topic ] ──► Consumer Group A (Analytics) — sees all messages
                      ──► Consumer Group B (Notifications) — sees all messages
                      ──► Consumer Group C (Audit Log) — sees all messages

Each consumer group gets its OWN copy of every message.
Independent consumption — each group tracks its own position.
```

```
Real example: User places order →
  ├── Inventory service (decrements stock)
  ├── Email service (sends confirmation)
  ├── Analytics service (tracks revenue)
  └── Fraud service (checks for anomalies)
  All from one "order.placed" event — zero coupling between services.
```

**Used by:** Kafka (consumer groups), SNS, RabbitMQ (fanout exchange)

---

### 4.3 Request-Reply (RPC over Queue)

> Async request-response. Producer sends a message and waits for a reply on a separate reply queue. Correlation ID ties request to response.

```
Client ──request──► [ Request Queue ] ──► Server
                                          │
Client ◄──response── [ Reply Queue ] ◄───┘

Message headers:
  reply_to: "queue.replies.client123"
  correlation_id: "req-abc-456"
```

**Used for:** Microservice RPC where you want async retry/durability but still need a response

---

### 4.4 Pattern Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│              WHEN TO USE WHICH PATTERN                               │
│                                                                      │
│  Point-to-Point (Task Queue)                                         │
│  ├── Each task needs to be done ONCE                                 │
│  ├── Load balance work across worker pool                            │
│  └── Use: RabbitMQ queue, SQS, Celery workers                        │
│                                                                      │
│  Pub/Sub (Fan-Out)                                                   │
│  ├── Multiple independent services react to same event               │
│  ├── New consumers can be added without producer changes             │
│  └── Use: Kafka topic, SNS → SQS fan-out, RabbitMQ fanout exchange   │
│                                                                      │
│  Request-Reply                                                       │
│  ├── Need response but want async durability                         │
│  └── Use: RabbitMQ with reply queues + correlation IDs               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Message Delivery Guarantees

> One of the most important concepts for interviews. Every messaging system makes a trade-off between safety, performance, and complexity.

### At-Most-Once

> Message is delivered **zero or one time**. If delivery fails, message is lost. Never duplicated.

```
Producer sends message
  → Consumer receives
  → Consumer immediately ACKs (before processing)
  → Consumer processes
  → If crash occurs between ACK and processing → message LOST 💥

Offset committed BEFORE processing.
```

```
When to accept this: Metrics, logs, analytics where losing a small
percentage of data is acceptable and performance matters most.
```

---

### At-Least-Once ⭐ Default for most systems

> Message is delivered **one or more times**. Guaranteed no loss. Duplicates possible.

```
Producer sends message
  → Consumer receives
  → Consumer processes
  → Consumer ACKs (after successful processing)
  → If crash occurs before ACK → broker re-delivers message
  → Consumer processes again → DUPLICATE ⚠️

Offset committed AFTER processing.
```

```
Handling duplicates: Implement IDEMPOTENT consumers
  (processing the same message twice = same result as processing once)
```

---

### Exactly-Once

> Message delivered and processed **exactly once**. No loss, no duplicates.

```
The hardest guarantee to achieve. Requires coordination between:
  - Idempotent producer (deduplicates at broker level)
  - Transactional writes (atomic offset commit + output write)
  - Idempotent consumer (deduplicates at app level)

Kafka: enable.idempotence=true + transactional API + isolation.level=read_committed
SQS FIFO: MessageDeduplicationId + ContentBasedDeduplication
```

```
Cost: Higher latency, more complexity, lower throughput.
Use only when you truly cannot tolerate duplicates (financial transactions).
```

### Delivery Guarantee Summary

| Guarantee | Loss Risk | Duplicate Risk | Complexity | Use Case |
|---|---|---|---|---|
| **At-Most-Once** | ⚠️ Yes | ❌ Never | Low | Metrics, logs |
| **At-Least-Once** | ❌ Never | ⚠️ Possible | Medium | Most systems (+ idempotency) |
| **Exactly-Once** | ❌ Never | ❌ Never | High | Financial, billing |

> **Default recommendation:** At-Least-Once + idempotent consumers. It's the sweet spot.

---

## 6. Kafka — Deep Dive

> Apache Kafka is a **distributed, append-only log**. It's the right choice for high-throughput event streaming, audit logs, real-time data pipelines, and replay scenarios.

### 6.1 Core Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         KAFKA CLUSTER                                   │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Broker 1   │    │   Broker 2   │    │   Broker 3   │              │
│  │              │    │              │    │              │              │
│  │ Topic: orders│    │              │    │              │              │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │              │
│  │ │Partition0│ │    │ │Partition1│ │    │ │Partition2│ │              │
│  │ │ (Leader) │ │    │ │ (Leader) │ │    │ │ (Leader) │ │              │
│  │ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │              │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │              │
│  │ │Part1     │ │    │ │Part2     │ │    │ │Part0     │ │              │
│  │ │(Replica) │ │    │ │(Replica) │ │    │ │(Replica) │ │              │
│  │ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│                                                                         │
│  ZooKeeper / KRaft (Kafka 4.0+): Cluster coordination, leader election  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 6.2 Topics, Partitions, and Offsets

**Topic:** A named logical channel. Like a database table but for events.

**Partition:** An ordered, immutable, append-only log segment of a topic.

```
Topic: "orders"  (3 partitions)

Partition 0:  [msg0] [msg1] [msg2] [msg3] [msg4] ...
               off=0  off=1  off=2  off=3  off=4

Partition 1:  [msg0] [msg1] [msg2] ...
               off=0  off=1  off=2

Partition 2:  [msg0] [msg1] [msg2] [msg3] ...
               off=0  off=1  off=2  off=3
```

Key properties:

- **Ordering guaranteed within a partition**, NOT across partitions
- **Offset** = integer position of message within a partition (immutable, never changes)
- Messages are **never deleted on read** — they stay until retention expires
- Messages are **append-only** — no updates, no deletes (except compaction)

**Partitioning logic (producer side):**

```python
# Explicit key → same key always goes to same partition (consistent hash)
producer.send("orders", key=b"user-42", value=order_json)
# hash("user-42") % num_partitions = partition index

# No key → round-robin across partitions
producer.send("orders", value=order_json)
```

> **Critical interview point:** Use a **partition key** when you need ordering for a specific entity (e.g., all events for `user-42` must be in order → key = user_id).

---

### 6.3 Consumer Groups — Horizontal Scaling

```
Topic "orders" with 3 partitions:
  Partition 0, Partition 1, Partition 2

Consumer Group "order-processors" (3 consumers):
  Consumer A → reads Partition 0 only
  Consumer B → reads Partition 1 only
  Consumer C → reads Partition 2 only

  ✅ Parallel processing — 3x throughput
  ✅ Each message processed by exactly ONE consumer in the group

Consumer Group "analytics" (1 consumer):
  Consumer D → reads ALL 3 partitions

  ✅ Completely independent of "order-processors" group
  ✅ Gets its own copy of all messages
```

```
RULES:
  Max parallelism = number of partitions
  If consumers > partitions → excess consumers are IDLE
  If consumers < partitions → some consumers read multiple partitions
  
  Scale out: Add more partitions + more consumers (can't reduce partitions)
```

**Consumer Lag:**

```
Lag = (Latest offset in partition) - (Consumer's committed offset)

Lag = 0        → Consumer is caught up ✅
Lag growing    → Consumer is too slow — scale up consumers ⚠️
Lag = millions → Consumer is falling behind — critical alert 🔴
```

---

### 6.4 Offsets and Delivery Semantics

> The offset is a **bookmark** per (consumer group, topic, partition) stored in an internal topic `__consumer_offsets`.

```
Partition 0: [0][1][2][3][4][5][6][7][8][9] ...
                              ↑
              committed_offset=4  (group has processed 0–3)

On restart: consumer resumes from offset 4
```

```python
# At-Least-Once (correct default):
while True:
    messages = consumer.poll(timeout=1.0)
    for msg in messages:
        process(msg)          # process FIRST
        consumer.commit()     # then commit offset
    # If crash before commit → same messages re-delivered → OK (idempotent)

# At-Most-Once (dangerous — rarely used):
while True:
    messages = consumer.poll(timeout=1.0)
    consumer.commit()         # commit FIRST
    for msg in messages:
        process(msg)          # If crash here → messages LOST
```

---

### 6.5 Replication — Fault Tolerance

```
Replication factor = 3  (each partition has 1 leader + 2 replicas)

Partition 0:
  Broker 1: LEADER (handles all reads/writes)
  Broker 2: Replica (follower, syncs from leader)
  Broker 3: Replica (follower, syncs from leader)

If Broker 1 fails:
  → Controller (KRaft / ZooKeeper) detects failure
  → Elects new leader from in-sync replicas (ISR)
  → Broker 2 becomes leader in ~seconds
  → Zero message loss (was in ISR)
```

**Producer durability config (`acks`):**

```python
# acks=0: Fire and forget — maximum throughput, possible loss
# acks=1: Leader acknowledges — fast, lose data if leader fails before replica sync
# acks=all (acks=-1): All ISR replicas ack — strongest durability, higher latency

producer = KafkaProducer(acks='all', retries=3, enable_idempotence=True)
```

---

### 6.6 Log Compaction

> Instead of deleting old messages by time/size, keep only the **latest value per key**. Deleted records marked with a **tombstone** (null value).

```
BEFORE compaction (topic: user-emails):
  offset 0: user-42 → alice@old.com
  offset 1: user-99 → bob@example.com
  offset 2: user-42 → alice@new.com    ← newer value for same key
  offset 3: user-42 → null             ← tombstone (delete)

AFTER compaction:
  offset 1: user-99 → bob@example.com  (kept — only value)
  offset 2: alice@new.com   (kept — latest for user-42... but)
  offset 3: user-42 → null  (tombstone — signals deletion)
  
  Eventually offset 3 removed too after tombstone retention window
```

**Use case:** Database changelog caching — rebuild current state from compacted log without replaying entire history. Used in Kafka Streams, CDC (Change Data Capture).

---

### 6.7 Kafka Config Reference

| Config | Value | Meaning |
|---|---|---|
| `replication.factor` | `3` | 3 copies of each partition |
| `acks` | `all` | Wait for all ISR replicas |
| `enable.idempotence` | `true` | Exactly-once producer |
| `retention.ms` | `604800000` | Keep messages for 7 days |
| `retention.bytes` | `-1` | No size limit (or set limit) |
| `auto.offset.reset` | `earliest` | New group starts from beginning |
| `enable.auto.commit` | `false` | Manual offset control (safer) |
| `max.poll.interval.ms` | `300000` | Max time between polls before rebalance |
| `isolation.level` | `read_committed` | For exactly-once consumers |
| `log.cleanup.policy` | `compact` | Enable log compaction |

---

## 7. RabbitMQ — Deep Dive

> RabbitMQ is a **message broker** built on AMQP. It excels at complex routing, traditional task queues, and scenarios where messages are consumed and deleted (not replayed).

### 7.1 Core Architecture

```
Producer ──► Exchange ──► (binding rules) ──► Queue ──► Consumer
              │
    Exchange decides WHERE the message goes based on routing key + exchange type
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   RABBITMQ ARCHITECTURE                         │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌─────────┐  ┌──────────┐  │
│  │ Producer │──►│   Exchange   │──►│  Queue  │─►│Consumer 1│  │
│  └──────────┘   │              │   └─────────┘  └──────────┘  │
│                 │  (routing     │   ┌─────────┐  ┌──────────┐  │
│                 │   logic here) │──►│  Queue  │─►│Consumer 2│  │
│                 └──────────────┘   └─────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 7.2 Exchange Types

**Direct Exchange** — Route by exact routing key match

```
Exchange: "order_exchange" (direct)
  Binding: routing_key="order.paid"     → Queue: payment-queue
  Binding: routing_key="order.shipped"  → Queue: shipping-queue

Message with routing_key="order.paid" → only payment-queue receives it
```

**Fanout Exchange** — Broadcast to ALL bound queues (ignores routing key)

```
Exchange: "events_fanout" (fanout)
  Bound queues: analytics-queue, notifications-queue, audit-queue

Any message → ALL 3 queues receive it
Use for: pub/sub broadcast
```

**Topic Exchange** — Route by routing key pattern (wildcards)

```
Exchange: "app_events" (topic)
  Binding: "order.*"    → Queue: order-queue      (matches order.paid, order.shipped)
  Binding: "user.#"     → Queue: user-queue       (matches user.created, user.profile.updated)
  Binding: "#"          → Queue: audit-queue      (matches everything)

  * = exactly one word
  # = zero or more words
```

**Headers Exchange** — Route by message header attributes (not routing key)

```
Message headers: {format: "json", type: "report"}
Binding: {format: "json"} → report-json-queue
Binding: {type: "report"} → all-reports-queue
```

---

### 7.3 Message Acknowledgement in RabbitMQ

```python
import pika

def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)   # ✅ success → delete from queue
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag,  # ❌ failure → requeue
                      requeue=True)

channel.basic_consume(queue='tasks', on_message_callback=callback)
```

**Durability config:**

```python
# Survive broker restart:
channel.queue_declare(queue='tasks', durable=True)  # durable queue
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body=message,
    properties=pika.BasicProperties(delivery_mode=2)  # persistent message
)
```

---

### 7.4 RabbitMQ vs Kafka Key Difference

| Aspect | RabbitMQ | Kafka |
|---|---|---|
| **Message lifecycle** | Deleted after ACK | Retained (configurable, default 7 days) |
| **Replay** | ❌ Not possible | ✅ Replay from any offset |
| **Routing** | Rich (exchanges, bindings) | Simple (topic + partition key) |
| **Ordering** | Per-queue | Per-partition |
| **Consumer model** | Push (broker pushes to consumer) | Pull (consumer polls broker) |
| **Primary use** | Task queues, RPC, complex routing | Event streaming, audit log, pipelines |

---

## 8. AWS SQS + SNS

### SQS — Simple Queue Service

> Fully managed message queue. No servers to manage, auto-scales, pay-per-use.

```
Standard Queue:
  ✅ Unlimited throughput
  ✅ At-least-once delivery
  ⚠️ Best-effort ordering (NOT guaranteed FIFO)
  ⚠️ Messages may be delivered more than once → design idempotent consumers

FIFO Queue:
  ✅ Exactly-once processing (with deduplication ID)
  ✅ Strict FIFO ordering within a message group
  ⚠️ 3,000 messages/sec with batching (vs unlimited for Standard)
  Use: Financial transactions, order processing with strict sequence
```

**Key SQS concepts:**

```
Visibility Timeout:
  Consumer reads message → message becomes INVISIBLE to others for N seconds
  Consumer ACKs (deletes) before timeout → message removed ✅
  Consumer fails / timeout expires → message becomes visible again → re-delivered

  ┌────────────────────────────────────────────────────────────┐
  │ Message read → [invisible 30s] → processed → delete ✅     │
  │                     OR                                     │
  │ Message read → [invisible 30s] → consumer crashes          │
  │             → timeout expires → visible again → re-queued  │
  └────────────────────────────────────────────────────────────┘

Long Polling:
  Consumer polls SQS, waits up to 20 seconds for messages (vs short poll: instant)
  Reduces empty responses → cheaper + lower latency
  config: WaitTimeSeconds=20

Message Retention: 1 minute to 14 days (default: 4 days)
Max Message Size: 256 KB (use S3 + pointer for larger payloads)
```

---

### SNS — Simple Notification Service

> Fully managed pub/sub. Publishers push to a **topic**, SNS fans out to all subscribers.

```
SNS Topic: "order-events"
  Subscriber 1: SQS queue (inventory-service)
  Subscriber 2: SQS queue (email-service)
  Subscriber 3: Lambda function (fraud-check)
  Subscriber 4: HTTP endpoint (webhook)

  One publish → all 4 subscribers notified simultaneously
```

### SNS + SQS Fan-Out Pattern ⭐

> Standard production pattern in AWS. SNS fans out to multiple SQS queues. Each service consumes from its own queue independently.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SNS + SQS FAN-OUT PATTERN                         │
│                                                                     │
│  Order Service                                                      │
│      │                                                              │
│      ▼ publish                                                      │
│  [ SNS Topic: order.placed ]                                        │
│      │                                                              │
│      ├──► [ SQS: inventory-queue ] ──► Inventory Service           │
│      ├──► [ SQS: email-queue     ] ──► Email Service               │
│      ├──► [ SQS: analytics-queue ] ──► Analytics Service           │
│      └──► [ SQS: fraud-queue     ] ──► Fraud Detection             │
│                                                                     │
│  Each SQS queue:                                                    │
│  - Has its own DLQ for failed messages                              │
│  - Can scale consumers independently                                │
│  - Adding new service = just add new SQS subscription to SNS       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Kafka vs RabbitMQ vs SQS

| Feature | **Kafka** | **RabbitMQ** | **AWS SQS** |
|---|---|---|---|
| **Type** | Distributed log / streaming | Message broker (AMQP) | Managed queue |
| **Throughput** | Millions msg/sec | ~50K–100K msg/sec | Unlimited (Standard) |
| **Latency** | Low (~1–5ms) | Very Low (~1ms) | Low–Medium (~10–50ms) |
| **Message Retention** | Time or size (days/weeks) | Until consumed | Up to 14 days |
| **Replay** | ✅ Yes (from any offset) | ❌ No | ❌ No |
| **Message Ordering** | Per partition | Per queue | FIFO queue only |
| **Routing** | Basic (topic + key) | Rich (exchange types) | None |
| **Delivery Guarantee** | At-least-once (exactly-once with config) | At-least-once | At-least-once (Standard) / Exactly-once (FIFO) |
| **Consumer Model** | Pull | Push | Pull (long poll) |
| **Scaling** | Add partitions + consumers | Add consumers | Auto (managed) |
| **Persistence** | On disk (durable) | Optional (durable flag) | Managed |
| **Ops Overhead** | High (cluster, ZooKeeper/KRaft) | Medium (cluster optional) | Zero (fully managed) |
| **Best For** | Event streaming, pipelines, audit log | Task queues, complex routing | Cloud-native AWS workloads |
| **AWS Equivalent** | Amazon MSK | Amazon MQ | SQS (native) |

### Decision Tree

```
Need to REPLAY messages or build event log?
  └── YES → Kafka

Need high throughput (> 100K msg/sec)?
  └── YES → Kafka

Need complex routing (topic/header matching)?
  └── YES → RabbitMQ

Building on AWS, want zero ops?
  └── YES → SQS (Standard or FIFO)

Need to fan out one event to many services on AWS?
  └── YES → SNS + SQS (fan-out pattern)

Simple task queue (one producer, one worker pool)?
  └── RabbitMQ or SQS (simplest)

Need exactly-once semantics?
  └── SQS FIFO or Kafka with transactions
```

---

## 10. Dead Letter Queues (DLQ)

> A DLQ is a special queue where messages are sent when they **cannot be successfully processed** after a configured number of retries. It prevents bad messages from blocking the queue forever.

```
Normal flow:
  Queue → Consumer → ✅ processed → deleted

Failure flow (no DLQ):
  Queue → Consumer → ❌ fails → requeued → Consumer → ❌ fails → poison loop 🔴

Failure flow (with DLQ):
  Queue → Consumer → ❌ fails → retry 1 → ❌ → retry 2 → ❌ → retry N
        → maxReceiveCount exceeded
        → message moved to DLQ ✅
        → main queue unblocked
        → engineers inspect DLQ for root cause
```

```
┌───────────────────────────────────────────────────────────────┐
│                    DLQ FLOW                                   │
│                                                               │
│  [ Main Queue ] ──► Consumer                                  │
│        │               │                                      │
│        │               ├── ✅ Success → ACK → deleted        │
│        │               └── ❌ Fail  → NACK → requeue         │
│        │                                │                    │
│        │         (after N retries)      │                    │
│        │◄──────────────────────────────┘                     │
│        │                                                      │
│  maxReceiveCount reached (e.g. 3):                           │
│        │                                                      │
│        ▼                                                      │
│  [ Dead Letter Queue ] ← parked here for inspection          │
│        │                                                      │
│        ├── Alert engineers                                    │
│        ├── Investigate root cause                             │
│        └── Replay after fix (SQS: redrive, Kafka: reset offset│
└───────────────────────────────────────────────────────────────┘
```

**DLQ config (SQS example):**

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:...:order-processing-dlq",
  "maxReceiveCount": 3
}
```

**Retry with Exponential Backoff:**

```
Attempt 1: immediate
Attempt 2: wait 1s
Attempt 3: wait 2s
Attempt 4: wait 4s
Attempt 5: → DLQ

Prevents hammering a struggling downstream service.
Add jitter to avoid synchronized retry storms.
```

---

## 11. Idempotency

> **Definition:** A consumer is idempotent when processing the same message multiple times produces the same result as processing it once. Required for at-least-once delivery systems.

With at-least-once delivery, duplicates happen. Your consumers must handle it.

```
Without idempotency:
  Message: "charge user-42 for $100"
  Delivered 3 times (consumer crashed twice before ACK)
  → User charged $300 🔴

With idempotency:
  Message: "charge user-42, idempotency_key: order-789"
  Delivered 3 times
  → Check: has order-789 been processed?
  → Attempt 1: No → process → store {order-789: processed}
  → Attempt 2: Yes → skip ✅
  → Attempt 3: Yes → skip ✅
  → User charged $100 exactly once ✅
```

### Implementation Patterns

#### Pattern 1: Idempotency Key in DB

```python
def process_order_payment(message):
    idempotency_key = message['order_id']

    # Check if already processed
    if db.exists(f"processed:{idempotency_key}"):
        return  # Skip duplicate

    # Process
    payment_service.charge(message['user_id'], message['amount'])

    # Mark as done (with TTL to avoid unbounded growth)
    db.setex(f"processed:{idempotency_key}", 86400, "1")
```

#### Pattern 2: Natural Idempotency (SET, not INCREMENT)

```sql
-- NOT idempotent (increment):
UPDATE accounts SET balance = balance - 100 WHERE id = 42;

-- Idempotent (set to absolute value):
UPDATE accounts SET balance = 400 WHERE id = 42 AND balance = 500;
-- If run twice → second update affects 0 rows (condition fails) → safe
```

---

## 12. Event-Driven Architecture (EDA)

> **Definition:** A software architecture where components communicate by producing and consuming events. Services are fully decoupled — producers don't know who listens; consumers don't know who produced.

```
TRADITIONAL (Request-Driven):
  OrderService ──HTTP──► InventoryService
               ──HTTP──► NotificationService
               ──HTTP──► AnalyticsService

  OrderService must know about all downstream services.
  Adding a new service = change OrderService.

EVENT-DRIVEN:
  OrderService ──publish──► [ "order.placed" event ]
                                      │
               ┌───────────────────────┤
               ▼                       ▼                  ▼
        InventoryService    NotificationService    AnalyticsService
         (subscribes)           (subscribes)         (subscribes)

  OrderService knows nothing about downstream services.
  Adding a new service = just subscribe, zero changes to OrderService. ✅
```

### Event Sourcing

> Instead of storing current state, store the **full history of events** that led to the current state. Current state = replay of all events.

```
TRADITIONAL state store:
  users table: { id: 42, balance: 450 }  ← current value only

EVENT SOURCING:
  events table:
    { user: 42, event: "account.created",  amount: 0,    ts: t1 }
    { user: 42, event: "deposit",          amount: 500,  ts: t2 }
    { user: 42, event: "withdrawal",       amount: 50,   ts: t3 }
    Current balance = 0 + 500 - 50 = 450

Benefits:
  ✅ Full audit trail — every state change is recorded
  ✅ Time travel — reconstruct state at any point in time
  ✅ Event replay — rebuild read models or fix bugs by replaying

Used by: Banking, e-commerce orders, Kafka-backed systems
```

### CQRS (Command Query Responsibility Segregation)

> Separate the write model (commands) from the read model (queries).

```
Write side:
  API → Command Handler → Event Store (Kafka) → Aggregate updated

Read side:
  Event Consumer → Builds read-optimised projections → Read DB (Elasticsearch, Redis)
  API → Query Handler → Read DB

Benefit: Write optimised for consistency, read optimised for speed.
Combined with Event Sourcing for full audit + fast reads.
```

---

## 13. Real-World Architectures

### Uber — Real-Time Dispatch (Kafka)

```
Driver GPS pings every 4 seconds → Kafka topic: "driver.location"
  ├── Dispatch Service (consumer group): matches drivers to riders
  ├── ETA Service: updates arrival time estimates
  ├── Analytics: stores for ML model training
  └── Surge Pricing: detects supply/demand imbalance

Scale: Millions of location events/minute
Why Kafka: High throughput, multiple consumer groups, replay for ML training
```

### Netflix — Keystone Pipeline (Kafka)

```
Every Netflix app event (play, pause, search, click) → Kafka
  → 500 billion+ events/day
  → Multiple consumers:
      Monitoring / alerting
      A/B test data
      Recommendation model training
      Business analytics

Topic: "viewing_activity"
Consumers can replay to rebuild analytics after a bug fix.
```

### E-Commerce Order Processing

```
User places order:

  Order API ──publish──► SNS: "order.placed"
                            │
             ┌──────────────┼──────────────────┐
             ▼              ▼                  ▼
      [SQS: inventory]  [SQS: payment]  [SQS: notifications]
             │              │                  │
      Inventory Svc    Payment Svc       Email/SMS Svc
      (decrements      (charges card)    (sends receipt)
       stock)

  All 3 services process independently.
  Each has its own DLQ.
  Payment failure → goes to payment-dlq → engineer notified.
  Order processing continues for inventory + notifications.
```

### Rate Limiting / Throttling Queue

```
Problem: 3rd party API allows 100 calls/sec. You have 10,000 calls to make.

Solution:
  Tasks → [ SQS Queue ] → Lambda consumers (max concurrency = 10)
                          Each makes ~10 API calls/sec
                          Total: 100 calls/sec → within rate limit ✅

  SQS acts as the buffer — Lambda scales to match queue depth.
```

---

## 14. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **No DLQ** | Bad messages cause infinite retry loops — block the queue | Always configure DLQ + `maxReceiveCount` |
| **No idempotency** | At-least-once delivery → duplicate processing → double charges, duplicate emails | Implement idempotency keys in every consumer |
| **Consumers > partitions (Kafka)** | Excess consumers are idle — wasted resources | Max parallelism = partition count |
| **Auto-commit offsets (Kafka)** | Crash between commit and process → data loss | `enable.auto.commit=false`, commit after processing |
| **Message too large** | Broker rejection, slowdowns | Cap at 1 MB or use S3 pointer for large payloads |
| **No backpressure handling** | Producer floods queue → consumer lag builds → memory exhaustion | Monitor lag; auto-scale consumers; apply producer throttle |
| **Global ordering assumption** | Expecting total order across partitions — Kafka doesn't guarantee this | Key messages by entity ID for per-entity ordering |
| **Synchronous calls inside consumer** | Slow downstream → consumer timeouts → rebalances | Use async I/O inside consumers; set `max.poll.interval.ms` carefully |
| **Not monitoring consumer lag** | Queue backs up silently until SLA breach | Alert on consumer lag > N messages |
| **Single queue / single partition** | No parallelism, throughput bottleneck | Design for multiple partitions from the start |

---

## 15. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Message Queue** | Durable async intermediary between producer and consumer |
| **Kafka** | Distributed append-only log for high-throughput event streaming |
| **RabbitMQ** | AMQP broker with rich routing for task queues |
| **SQS** | Fully managed AWS queue, zero ops overhead |
| **Consumer Group** | Kafka: group of consumers splitting a topic's partitions |
| **Offset** | Kafka: integer bookmark of message position in partition |
| **DLQ** | Queue for messages that repeatedly fail — for inspection/replay |
| **At-Least-Once** | No message loss, duplicates possible — use idempotency |
| **Exactly-Once** | No loss, no duplicates — requires transactions (complex) |
| **Idempotency** | Same message processed twice = same outcome as once |
| **Consumer Lag** | Gap between latest produced and latest consumed offset |
| **Log Compaction** | Kafka: keep only latest value per key, discard old versions |
| **SNS + SQS Fan-Out** | SNS broadcasts to multiple SQS queues — AWS pub/sub pattern |
| **Event Sourcing** | Store events, not state — current state = replay of events |

### When to Use What (Scenarios)

| Scenario | Recommendation |
|---|---|
| High-throughput event pipeline (> 100K msg/sec) | **Kafka** |
| Need to replay messages / audit log | **Kafka** |
| Multiple independent services react to same event | **Kafka** (consumer groups) or **SNS + SQS** |
| Complex routing (headers, patterns, fanout) | **RabbitMQ** (topic/fanout exchange) |
| Simple task queue, worker pool | **RabbitMQ** or **SQS Standard** |
| AWS cloud-native, zero ops | **SQS + SNS** |
| Strict FIFO + exactly-once (financial) | **SQS FIFO** or **Kafka transactions** |
| Background job processing | **SQS** + Lambda / **RabbitMQ** + Celery |
| Rate limiting calls to external API | **SQS** as buffer + controlled consumer concurrency |
| Microservice event bus | **Kafka** (large scale) or **SNS + SQS** (AWS) |
| Change Data Capture (DB → downstream) | **Kafka** + Debezium |

### The Interview Answer Template

```
1. WHY a queue here:
   "I'll use a message queue to decouple [OrderService] from 
    [InventoryService, EmailService] — they can scale and fail 
    independently, and we get a buffer for traffic spikes."

2. WHICH technology:
   "Since this is on AWS, I'd use SNS + SQS fan-out.
    For event streaming at scale / replay, I'd use Kafka."

3. DELIVERY guarantee:
   "At-least-once — the default and simplest to implement.
    Consumers will be idempotent using an idempotency key
    stored in Redis with a 24-hour TTL."

4. FAILURE handling:
   "Each SQS queue has a DLQ with maxReceiveCount=3.
    Failed messages parked in DLQ → CloudWatch alert fires
    → engineer investigates → redrive after fix."

5. SCALING:
   "Consumer lag is our key metric. Auto-scaling group 
    scales consumer EC2 instances when lag exceeds threshold.
    For Kafka: pre-provision enough partitions to support
    the target consumer count."
```

### Must-Know Interview Points

- ☑ **At-Least-Once + idempotency** is the practical default. Know how to implement it.
- ☑ **Kafka ordering: per-partition only.** To order events for a user → key by user_id.
- ☑ **Consumer count ≤ partition count** in Kafka. Excess consumers are idle.
- ☑ **DLQ is mandatory** in any production queue. Always mention it.
- ☑ **Commit offset AFTER processing**, not before. Auto-commit is dangerous.
- ☑ **SNS + SQS** is the standard AWS fan-out pattern. Know it cold.
- ☑ **Consumer lag** is the health metric for queues. Alert on growth.
- ☑ **RabbitMQ deletes on ACK. Kafka retains.** This drives their different use cases.
- ☑ **Queue = buffer for traffic spikes.** Always mention this in scalability discussions.
- ☑ **Event Sourcing + CQRS** → mention for audit trails and high read/write ratio systems.

---

*Sources: AlgoMaster.io, DEV Community (System Design EDA), HelloInterview Kafka Deep Dive, Confluent Documentation (Consumer Design, Delivery Semantics, Log Compaction), DesignGurus, Calmops Message Queue Deep Dive, Conduktor Kafka Consumer Groups, galfrevn.com Queue Messaging — combined with first-principles system design knowledge.*