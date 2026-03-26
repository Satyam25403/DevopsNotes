# 🧩 System Design Patterns

> **Series:** System Design Notes  
> **Module:** 11 — System Design Patterns  
> **Prerequisites:** `07_message_queues.md`, `08_api_design.md`, `05_scalability.md`, Basic microservices concepts

---

## 📌 Table of Contents

**Resilience Patterns**
1. [Circuit Breaker](#1-circuit-breaker)
2. [Retry + Exponential Backoff](#2-retry--exponential-backoff)
3. [Bulkhead](#3-bulkhead)
4. [Timeout](#4-timeout)
5. [Fallback & Graceful Degradation](#5-fallback--graceful-degradation)

**Data Patterns**
6. [SAGA Pattern](#6-saga-pattern)
7. [Transactional Outbox](#7-transactional-outbox)
8. [CQRS](#8-cqrs-command-query-responsibility-segregation)
9. [Event Sourcing](#9-event-sourcing)

**Infrastructure Patterns**
10. [Sidecar](#10-sidecar)
11. [Service Mesh](#11-service-mesh)
12. [Service Discovery](#12-service-discovery)

**Migration Patterns**
13. [Strangler Fig](#13-strangler-fig)
14. [Shadow Deployment & Blue-Green](#14-shadow-deployment--blue-green)

**Reference**
15. [Pattern Decision Matrix](#15-pattern-decision-matrix)
16. [Interview Cheatsheet](#16-interview-cheatsheet)

---

## RESILIENCE PATTERNS

> Resilience patterns keep your system alive when parts of it fail. In a distributed system, **partial failure is the norm, not the exception.** These patterns assume failure will happen and design around it.

```
The core principle: FAIL FAST, FAIL SAFE, RECOVER GRACEFULLY

Don't: Let one slow service cascade into a full system outage
Do:   Detect failure early, contain the blast radius, degrade gracefully
```

---

## 1. Circuit Breaker

> **Problem:** Service A calls Service B. Service B is slow or down. Service A's threads pile up waiting → Service A runs out of threads → Service A goes down → cascades upward. One bad service brings down everything.

> **Solution:** Wrap calls to Service B in a circuit breaker. After N failures, "trip" the circuit — immediately return an error instead of waiting. Give B time to recover. Periodically probe to see if it's healthy.

### States

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CIRCUIT BREAKER STATE MACHINE                    │
│                                                                     │
│   ┌──────────┐   N failures    ┌──────────┐                        │
│   │  CLOSED  │────────────────►│   OPEN   │                        │
│   │(normal)  │                 │(failing) │                        │
│   └──────────┘                 └──────────┘                        │
│        ▲                            │                              │
│        │                            │ timeout expires              │
│        │                            ▼                              │
│        │  success            ┌──────────────┐                      │
│        └────────────────────-│  HALF-OPEN   │                      │
│          probe succeeds      │ (1 test req) │                      │
│                              └──────────────┘                      │
│                                     │                              │
│                                     │ probe fails                  │
│                                     ▼                              │
│                               back to OPEN                         │
└─────────────────────────────────────────────────────────────────────┘

CLOSED:    Normal operation. Calls flow through. Failure counter increments on each error.
           If failures exceed threshold (e.g. 5 in 10s) → trip to OPEN.

OPEN:      All calls IMMEDIATELY return error/fallback. No requests sent to Service B.
           After timeout (e.g. 30s) → move to HALF-OPEN.

HALF-OPEN: Allow ONE test request through to B.
           ✅ Success → back to CLOSED (B has recovered)
           ❌ Failure → back to OPEN (B still broken, wait again)
```

### What Happens in OPEN State (Fallback Options)

```
Request comes in, circuit is OPEN:
  Option 1: Return cached/stale data           ← best UX
  Option 2: Return default/empty response      ← safe default
  Option 3: Return error with Retry-After      ← honest failure
  Option 4: Route to backup service            ← redundancy
```

### Implementation (Resilience4j — Java)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // trip at 50% failure rate
    .slidingWindowSize(10)              // over last 10 calls
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .permittedNumberOfCallsInHalfOpenState(3)
    .build();

CircuitBreaker cb = CircuitBreaker.of("payment-service", config);

// Wrap the call
Supplier<PaymentResponse> decorated = CircuitBreaker
    .decorateSupplier(cb, () -> paymentService.charge(amount));

Try<PaymentResponse> result = Try.ofSupplier(decorated)
    .recover(CallNotPermittedException.class, ex -> fallbackResponse());
```

### Config Reference

| Parameter | Typical Value | Meaning |
|---|---|---|
| `failureRateThreshold` | 50% | Trip when 50%+ of calls fail |
| `slidingWindowSize` | 10–20 calls | Measure rate over this many calls |
| `waitDurationInOpenState` | 10–60s | Time to wait before half-open probe |
| `permittedCallsInHalfOpen` | 1–5 | Test calls before declaring recovery |
| `slowCallRateThreshold` | 80% | Also trip on slow calls (not just errors) |
| `slowCallDurationThreshold` | 2s | What counts as "slow" |

### Tools

| Tool | Language | Notes |
|---|---|---|
| **Resilience4j** | Java | Lightweight, functional, Spring Boot ready |
| **Hystrix** | Java | Netflix OSS, now in maintenance mode |
| **Polly** | .NET | Full resilience library |
| **Envoy** / **Istio** | Service mesh | Circuit breaking at proxy layer (no code) |
| **AWS API Gateway** | Managed | Built-in throttling + circuit breaking |

> **Netflix's origin story:** Netflix Hystrix was created after a cascading failure took down their entire platform. Every service now has a circuit breaker. If Recommendations fails → movies still play. If Reviews fails → movie details still load.

---

## 2. Retry + Exponential Backoff

> **Problem:** Many failures in distributed systems are transient — a network blip, a brief DB overload, a timeout. Immediately failing the user is unnecessary if a retry would succeed.

> **Solution:** Automatically retry failed operations, with increasing delays between attempts. Add jitter to prevent all retriers from syncing up and causing a retry storm.

### Backoff Strategies

```
FIXED INTERVAL (naive — don't use):
  Attempt 1: fail → wait 1s
  Attempt 2: fail → wait 1s
  Attempt 3: fail → wait 1s
  Problem: all clients retry at same time → synchronized thundering herd

EXPONENTIAL BACKOFF (better):
  Attempt 1: fail → wait 1s
  Attempt 2: fail → wait 2s
  Attempt 3: fail → wait 4s
  Attempt 4: fail → wait 8s
  Formula: wait = base * (2 ^ attempt_number)

EXPONENTIAL BACKOFF + JITTER (production standard ✅):
  Attempt 1: fail → wait 1s  + rand(0, 0.5s)  = 1.3s
  Attempt 2: fail → wait 2s  + rand(0, 1s)    = 2.7s
  Attempt 3: fail → wait 4s  + rand(0, 2s)    = 5.1s
  Attempt 4: fail → wait 8s  + rand(0, 4s)    = 10.2s
  Formula: wait = min(cap, base * (2^n)) + rand(0, base * (2^n))
```

```python
import random, time

def retry_with_backoff(fn, max_attempts=4, base_delay=1.0, cap=30.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise  # exhausted retries
            
            # Exponential backoff with full jitter
            delay = min(cap, base_delay * (2 ** attempt))
            jitter = random.uniform(0, delay)
            time.sleep(jitter)
    
    raise MaxRetriesExceeded()
```

### What to Retry vs Not

```
✅ SAFE TO RETRY (idempotent or transient):
  - 429 Too Many Requests (with Retry-After)
  - 502 Bad Gateway
  - 503 Service Unavailable
  - 504 Gateway Timeout
  - Network timeouts (if operation is idempotent)
  - Idempotent mutations (with idempotency key)

❌ DO NOT RETRY:
  - 400 Bad Request (client error — retry won't help)
  - 401 Unauthorized (fix auth, not retry)
  - 403 Forbidden (permission issue — retry won't help)
  - 404 Not Found (data doesn't exist — retry won't help)
  - 409 Conflict (logic error — retry won't help)
  - Non-idempotent POST without idempotency key (double charge risk!)
```

### Retry Budget

> Don't let retries amplify load during an outage. If upstream is failing for all services, uncapped retries make the outage worse.

```
Retry budget: limit total retries across all callers to N% of requests.
If budget exhausted → fail fast, don't retry.

Example: Service handles 10,000 req/s
  Normal retries: ~100/s (1%)
  During outage: don't allow > 500 retries/s (5%)
  Beyond that → return error immediately
```

---

## 3. Bulkhead

> **Problem:** All services share the same thread pool or connection pool. One slow dependency consumes all threads → other (healthy) operations starve → total service failure.

> **Solution:** Isolate resources per dependency or workload type. Each gets its own thread pool / semaphore / connection pool. Failure in one pool can't drain resources from others.

> **Origin:** Named after ship bulkheads — watertight compartments that prevent a breach in one section from sinking the entire vessel.

### Thread Pool Bulkhead

```
WITHOUT BULKHEAD:
  Shared thread pool: [T1][T2][T3][T4][T5][T6][T7][T8]
  
  PaymentService slow → 8 threads waiting on payment
  InventoryService calls → no threads available → fail ❌
  UserService calls → no threads available → fail ❌
  Everything fails because of one slow dependency.

WITH BULKHEAD:
  Payment pool:   [T1][T2]       max=2, queue=5
  Inventory pool: [T3][T4][T5]   max=3, queue=10
  User pool:      [T6][T7][T8]   max=3, queue=10

  PaymentService slow → only T1, T2 blocked
  Inventory + User still have their own threads → unaffected ✅
```

```java
// Resilience4j Thread Pool Bulkhead
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(2)         // max concurrent calls to this dependency
    .coreThreadPoolSize(2)
    .queueCapacity(5)             // queue before rejecting
    .keepAliveDuration(Duration.ofMillis(20))
    .build();

ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("payment-service", config);
```

### Semaphore Bulkhead

```
Simpler, same-thread. Limits concurrent calls via a counter.
No separate thread pool — blocking happens in caller thread.

SemaphoreBulkheadConfig config = SemaphoreBulkheadConfig.custom()
    .maxConcurrentCalls(10)    // max 10 concurrent in-flight calls
    .maxWaitDuration(Duration.ofMillis(500))   // timeout if semaphore unavailable
    .build();
```

### Bulkhead in Practice (Netflix)

```
Netflix architecture: Each backend dependency gets isolated Hystrix thread pool.

User request arrives → calls 6 backend services:
  ┌────────────────────────────────────────┐
  │  [Recommendations: 5 threads max]      │
  │  [Ratings: 5 threads max]              │
  │  [VideoPlayback: 20 threads max] ←critical
  │  [UserProfile: 10 threads max]         │
  │  [Billing: 3 threads max]              │
  │  [Search: 8 threads max]               │
  └────────────────────────────────────────┘

Recommendations pool exhausted → recommendations unavailable
VideoPlayback unaffected → movies still play ✅
```

---

## 4. Timeout

> **Problem:** A call to a downstream service hangs indefinitely. Caller thread is stuck. Thread pool fills up. Service becomes unresponsive.

> **Solution:** Set explicit timeouts on every outbound call. Never allow indefinite blocking.

```
WITHOUT TIMEOUT:
  Service A calls Service B → B hangs → A waits forever
  A's thread pool: [waiting][waiting][waiting][waiting] → full → A is down

WITH TIMEOUT:
  Service A calls Service B with timeout=2s
  B hasn't responded in 2s → A receives TimeoutException
  A's thread freed → can serve other requests
  A returns fallback/503 to client
```

### Timeout Layering

```
Set timeouts at EVERY layer:

  HTTP client timeout:           connect=1s, read=5s
  Load balancer timeout:         idle=60s
  API Gateway timeout:           integration=29s (AWS Lambda max)
  Database query timeout:        statement=10s, connection=5s
  gRPC deadline:                 10s (propagated through call chain)
  External API call:             connect=2s, read=10s
  
The rule: outer timeout > sum of inner timeouts
Don't let the outer layer timeout before inner retries finish.
```

```python
import requests

response = requests.get(
    "http://payment-service/charge",
    timeout=(2, 10)   # (connect_timeout, read_timeout) in seconds
)
```

### Deadline Propagation (gRPC pattern)

```
Client sets overall deadline: 5s

API Gateway receives request → remaining time = 4.8s
  → Forwards to Order Service with deadline = 4.8s
      → Calls Payment Service with deadline = 3.1s (accounting for processing)
          → Calls Fraud Service with deadline = 1.5s
              If Fraud takes > 1.5s → Timeout! Error propagates back up.

No service can accidentally exceed the user-visible timeout.
```

---

## 5. Fallback & Graceful Degradation

> **Pattern:** When a service fails (circuit open, timeout, error), return something useful instead of nothing.

```
FAIL HARD (bad):
  User opens Netflix, Recommendations service is down
  → Show 500 error page
  → User leaves

GRACEFUL DEGRADATION (good):
  User opens Netflix, Recommendations service is down
  → Show "Popular Movies" (static list from cache)
  → User doesn't notice
  → Business impact: minimal
```

### Fallback Strategies

```
1. CACHED DATA:
   Return last known good response from cache.
   "Stale but not broken."
   Best for: Data that changes slowly (product catalogue, user prefs)

2. DEFAULT / EMPTY RESPONSE:
   Return a sensible default.
   "Empty but functional."
   Example: Empty recommendations [], 0 unread notifications

3. STATIC FALLBACK:
   Return hardcoded "safe" data.
   Example: "Sorry, reviews are temporarily unavailable"
   Best for: Non-critical features

4. ALTERNATE SERVICE / REPLICA:
   Route to a backup instance or read replica.
   Best for: Critical services with redundancy

5. QUEUE FOR LATER:
   Accept the request, queue it, process when service recovers.
   Best for: Non-time-sensitive operations (email, async updates)
```

---

## DATA PATTERNS

---

## 6. SAGA Pattern

> **Problem:** Distributed transactions across multiple microservices with separate databases. Traditional 2PC (Two-Phase Commit) is slow, blocking, and violates availability in distributed systems. When an order requires updates to Order DB, Inventory DB, and Payment DB — how do you ensure all-or-nothing atomicity?

> **Solution:** Break the transaction into a sequence of local transactions, each within a single service. If any step fails, execute **compensating transactions** to undo previous steps.

```
TRADITIONAL 2PC (Two-Phase Commit):
  Coordinator: "Prepare to commit" → all services lock their rows
  Coordinator: "Commit!"          → all commit simultaneously
  
  Problem: All services blocked while waiting. Network partition → deadlock.
  Violates CAP theorem availability. Not practical at microservice scale.

SAGA:
  T1 (Order Service):     Create order (PENDING state)
  T2 (Inventory Service): Reserve stock
  T3 (Payment Service):   Charge card
  T4 (Order Service):     Confirm order (CONFIRMED state)
  
  If T3 fails (card declined):
  C2 (Inventory Service): Release reserved stock    ← compensating transaction
  C1 (Order Service):     Cancel order              ← compensating transaction
  
  No locks held. Eventually consistent. ✅
```

### 6.1 Choreography-Based Saga

> Decentralized. Each service listens for events and independently reacts. No central controller.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   CHOREOGRAPHY SAGA — ORDER FLOW                       │
│                                                                        │
│  User places order                                                     │
│       ↓                                                                │
│  Order Service → creates order (PENDING) → publishes "OrderCreated"   │
│       ↓                                                                │
│  Inventory Service (listens) → reserves stock → publishes "StockReserved"
│       ↓                                                                │
│  Payment Service (listens) → charges card → publishes "PaymentSuccess"│
│       ↓                                                                │
│  Order Service (listens) → sets order CONFIRMED ✅                     │
│                                                                        │
│  FAILURE (Payment declined):                                           │
│  Payment Service → publishes "PaymentFailed"                           │
│       ↓                                                                │
│  Inventory Service (listens to PaymentFailed) → releases stock         │
│       ↓                                                                │
│  Order Service (listens to PaymentFailed) → cancels order              │
└────────────────────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Fully decoupled — services don't know each other | Hard to track overall transaction state (distributed) |
| No single point of failure | Cyclic dependencies can cause deadlocks |
| Easy to add new services (subscribe to events) | Debugging is difficult — trace spans multiple services |
| High scalability | Global retry/timeout policies harder to enforce |

**Best for:** Simple, high-volume, loosely coupled workflows. Notification flows. Event-driven systems.

---

### 6.2 Orchestration-Based Saga

> Centralized. A single **Orchestrator** service manages the entire flow — tells each service what to do, handles failures, triggers compensating transactions.

```
┌────────────────────────────────────────────────────────────────────────┐
│                  ORCHESTRATION SAGA — ORDER FLOW                       │
│                                                                        │
│  User places order                                                     │
│       ↓                                                                │
│  ┌──────────────────────────────────────┐                              │
│  │         ORDER ORCHESTRATOR           │  ← central brain             │
│  │                                      │                              │
│  │  Step 1: → Order Service             │                              │
│  │  Step 2: → Inventory Service         │                              │
│  │  Step 3: → Payment Service           │                              │
│  │  Step 4: → Order Service (confirm)   │                              │
│  │                                      │                              │
│  │  On Payment failure:                 │                              │
│  │    Compensate 2: → Inventory Service │  (release stock)             │
│  │    Compensate 1: → Order Service     │  (cancel order)              │
│  └──────────────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Single source of truth for transaction state | Orchestrator = single point of failure (mitigate with HA) |
| Easy to visualize, debug, and monitor | Orchestrator can become a bottleneck |
| Compensations are explicit and controlled | More coupling — orchestrator knows all participants |
| Retry/timeout policies in one place | Additional service to build and maintain |

**Best for:** Complex workflows, regulated processes (payments, refunds), workflows with many failure modes.

---

### 6.3 Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|---|---|---|
| **Control** | Decentralized (events) | Centralized (orchestrator) |
| **Coupling** | Loose | Tighter (orchestrator knows all) |
| **Debuggability** | Hard | Easy (central state) |
| **SPOF** | No | Orchestrator (mitigate with HA) |
| **Error handling** | Per service | Centralized |
| **Best for** | Simple, high-volume | Complex, stateful, regulated |
| **AWS implementation** | SNS + SQS | Step Functions |
| **Tools** | Kafka, Axon | AWS Step Functions, Temporal, Camunda |

> **Production pattern:** Mix both. Orchestration for money-moving steps (payment, refunds). Choreography for downstream side effects (emails, analytics, recommendations).

---

## 7. Transactional Outbox

> **Problem:** Service needs to update its database AND publish an event. These are two separate operations. If the DB write succeeds but the message publish fails (or vice versa) → inconsistent state (ghost record / missed event).

```
The dual-write problem:
  BEGIN TRANSACTION
    INSERT INTO orders (...)          ← DB write succeeds
  COMMIT
  
  kafka.publish("order.created", ...) ← Message publish FAILS 💥
  
  Result: Order exists in DB, but downstream services never notified.
  Inventory not updated. Email never sent. Silent inconsistency.
```

> **Solution:** Write the event to an **outbox table** in the SAME DB transaction. A background process (relay) reads the outbox and publishes to the message broker. Atomicity guaranteed by the DB transaction.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   TRANSACTIONAL OUTBOX PATTERN                      │
│                                                                     │
│  Service (DB Transaction):                                          │
│  ┌─────────────────────────────────────────┐                       │
│  │  INSERT INTO orders (...)               │   both in one         │
│  │  INSERT INTO outbox (event, payload)    │ ← DB transaction ✅   │
│  └─────────────────────────────────────────┘                       │
│                │                                                    │
│                │ (atomic — either both commit, or both rollback)    │
│                │                                                    │
│         Outbox Table:                                               │
│         id | event           | payload      | published            │
│         1  | order.created   | {...}        | false                │
│                │                                                    │
│         Outbox Relay (background process / Debezium):               │
│                │ polls outbox WHERE published=false                 │
│                │ → publishes to Kafka/SNS                          │
│                │ → marks published=true                             │
│                                                                    │
│         Kafka Consumer:                                             │
│                └──► Inventory Service                              │
│                └──► Email Service                                  │
└─────────────────────────────────────────────────────────────────────┘
```

**Relay implementation options:**
- **Polling:** Background job reads outbox every N seconds (simple, slight delay)
- **Debezium (CDC):** Reads DB binlog changes in real-time, publishes to Kafka (near-realtime, no polling overhead)

**Used by:** Every production event-driven system that needs reliable event publishing.

---

## 8. CQRS (Command Query Responsibility Segregation)

> **Problem:** Same data model must serve both heavy writes and heavy reads. Writes need normalization and ACID. Reads need denormalized, optimized projections — often from multiple entities joined together. Competing requirements kill performance.

> **Solution:** Separate the **write model** (commands) from the **read model** (queries). Each is optimized independently.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CQRS ARCHITECTURE                             │
│                                                                      │
│  WRITE SIDE (Commands):                                              │
│  Client → Command API → Command Handler → Write DB (normalized SQL)  │
│                              │                                       │
│                              │ publish event                         │
│                              ▼                                       │
│                     Message Broker (Kafka)                           │
│                              │                                       │
│                              │ consume event                         │
│                              ▼                                       │
│  READ SIDE (Queries):                                                │
│  Event Consumer → Build Projection → Read DB (Redis / Elasticsearch) │
│                                                                      │
│  Client → Query API → Read DB → Response (pre-aggregated, fast)     │
└──────────────────────────────────────────────────────────────────────┘
```

### Example: E-Commerce Order Dashboard

```
Write model (normalized PostgreSQL):
  orders table: id, user_id, status, created_at
  order_items table: order_id, product_id, qty, price
  products table: id, name, price

Read model (denormalized — e.g. Elasticsearch / Redis):
  "order_summary:user-42": {
    "recent_orders": [...],          ← pre-joined
    "total_spent": 1450.00,          ← pre-aggregated
    "favorite_category": "Electronics"  ← pre-computed
  }

  Query: GET /users/42/dashboard → Redis read → <1ms
  No joins, no aggregations at query time.
```

| ✅ Pros | ❌ Cons |
|---|---|
| Read model optimised independently (Redis, ES) | Eventual consistency between write and read |
| Write model optimised for transactions | More infrastructure to maintain |
| Scale reads and writes independently | Data duplication |
| Read projections rebuild-able from event log | Complexity — two models to keep in sync |

**Best for:** High read/write asymmetry, dashboards, reporting, search, timelines.

---

## 9. Event Sourcing

> **Problem:** You store only current state. You lose the history of HOW you got there — no audit trail, no ability to replay, no time travel.

> **Solution:** Store every state change as an immutable **event** in an append-only log. Current state = replay of all events. Never update in place.

```
TRADITIONAL STATE STORE:
  users table: { id: 42, balance: 450 }
  → Only current value. No history. Can't answer: 
    "what was the balance on Jan 1?" 
    "who changed it and when?"

EVENT STORE:
  events table:
    { user: 42, type: "AccountCreated",  data: {balance: 0},    ts: t1 }
    { user: 42, type: "MoneyDeposited",  data: {amount: 500},   ts: t2 }
    { user: 42, type: "MoneyWithdrawn",  data: {amount: 50},    ts: t3 }
    
  Current state:
    Replay all events → 0 + 500 - 50 = 450 ✅
    
  Time travel:
    Replay up to t2 → balance was 500 at that point ✅
    
  Audit log:
    Full history of every change, by whom, when ✅
```

```
Event Sourcing flow:
  Command arrives → Validate → Generate Event → Append to store
                                    ↓
                            Project to read model (CQRS)
                            Update materialized views
```

| ✅ Pros | ❌ Cons |
|---|---|
| Complete audit trail | Eventual consistency on projections |
| Time travel / debug at any point | Rebuilding state = replaying all events (can be slow) |
| Rebuild read models by replaying | Event schema migration is hard |
| Natural fit with CQRS | New concept — steep learning curve |
| Foundation for reliable messaging (Outbox) | Storage grows unbounded (use snapshots) |

**Snapshot optimization:**

```
After N events, save a snapshot of current state.
On rebuild: load latest snapshot + replay only events after it.

events: [e1][e2]...[e1000] + snapshot@1000 + [e1001][e1002]...
Rebuild: start from snapshot@1000, apply e1001+ only. Fast. ✅
```

**Used by:** Banking (every transaction is an event), e-commerce (order lifecycle), Kafka-backed systems, git (every commit is an event on a repo).

---

## INFRASTRUCTURE PATTERNS

---

## 10. Sidecar

> **Problem:** Every microservice needs cross-cutting concerns: TLS, logging, tracing, metrics, service discovery, retries, circuit breaking. Implementing these in each service is duplicated effort and inconsistent across languages/frameworks.

> **Solution:** Deploy a **sidecar process** alongside every service instance (in the same pod/VM). The sidecar handles all cross-cutting concerns. The main service only does its business logic.

```
┌──────────────────────────────────────────────────────┐
│                    KUBERNETES POD                    │
│                                                      │
│  ┌─────────────────┐    ┌─────────────────────────┐  │
│  │  Main Service   │    │       SIDECAR           │  │
│  │  (business      │◄──►│  (infrastructure proxy) │  │
│  │   logic only)   │    │                         │  │
│  │                 │    │  ✅ TLS termination      │  │
│  │  No retry code  │    │  ✅ Circuit breaking     │  │
│  │  No TLS code    │    │  ✅ Retries              │  │
│  │  No metrics code│    │  ✅ Metrics collection   │  │
│  │  No tracing code│    │  ✅ Distributed tracing  │  │
│  └─────────────────┘    │  ✅ mTLS enforcement     │  │
│                         │  ✅ Rate limiting        │  │
│  Any language!          └─────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

### Traffic Flow with Sidecar

```
Incoming request:
  External → Sidecar (validates JWT, terminates TLS) → Main Service

Outgoing request:
  Main Service → Sidecar (adds retry, circuit breaking, traces) → Target Service's Sidecar → Target Main Service
```

### Benefits

| Benefit | Details |
|---|---|
| **Language agnostic** | Sidecar handles infra in any language — Java, Python, Go, etc. |
| **Consistent policies** | Same retry/timeout/TLS config across ALL services |
| **Zero main service changes** | Deploy new sidecar version without touching business logic |
| **Centralized observability** | All traces/metrics go through sidecar → unified dashboard |

**Implementations:** Envoy (Istio/Linkerd uses it), Dapr, NGINX, HAProxy

---

## 11. Service Mesh

> A **service mesh** is when you deploy sidecars on EVERY service instance in a cluster, and connect them with a centralized control plane. The sidecars form a "mesh" of intelligent network proxies.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SERVICE MESH                                    │
│                                                                        │
│  CONTROL PLANE (Istiod / Linkerd controller):                          │
│  - Distributes config to all sidecars                                  │
│  - Manages certificates (mTLS)                                         │
│  - Defines traffic policies                                            │
│  - Aggregates telemetry                                                │
│                                                                        │
│  DATA PLANE (Envoy sidecars on every pod):                             │
│                                                                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│  │Service A     │    │Service B     │    │Service C     │             │
│  │┌──────────┐  │    │┌──────────┐  │    │┌──────────┐  │             │
│  ││ App      │  │    ││ App      │  │    ││ App      │  │             │
│  │└──────────┘  │    │└──────────┘  │    │└──────────┘  │             │
│  │┌──────────┐  │    │┌──────────┐  │    │┌──────────┐  │             │
│  ││ Sidecar  │◄─┼────┼►Sidecar  │◄─┼────┼►Sidecar  │  │             │
│  │└──────────┘  │    │└──────────┘  │    │└──────────┘  │             │
│  └──────────────┘    └──────────────┘    └──────────────┘             │
│                                                                        │
│  All traffic flows through sidecars → automatic mTLS, tracing, retry  │
└────────────────────────────────────────────────────────────────────────┘
```

### What You Get "For Free" with a Service Mesh

```
mTLS:          All service-to-service traffic encrypted + mutually authenticated
               No code changes. Control plane rotates certs automatically.

Observability: Every request traced. Golden signals (latency/traffic/errors/saturation)
               collected from every service automatically.

Traffic mgmt:  Canary deployments (route 5% to v2, 95% to v1)
               A/B testing, fault injection, traffic mirroring.

Resilience:    Retry, circuit breaking, timeout policies via config (no code).

Zero-trust:    Enforce that Service A can ONLY talk to Service B — via policy.
```

| Tool | Notes |
|---|---|
| **Istio** | Most feature-rich, uses Envoy sidecar, complex |
| **Linkerd** | Lightweight, simpler than Istio, Rust-based proxy |
| **Consul Connect** | HashiCorp, works outside Kubernetes too |
| **AWS App Mesh** | Managed, Envoy-based, AWS native |

---

## 12. Service Discovery

> **Problem:** Microservices are ephemeral. IPs change constantly (container restarts, scaling, rolling deploys). How does Service A find the current IP of Service B?

> **Solution:** A **service registry** where services register themselves on startup and deregister on shutdown. Clients query the registry to find healthy instances.

```
┌──────────────────────────────────────────────────────────────┐
│                    SERVICE DISCOVERY                         │
│                                                              │
│  Service B starts → registers with registry                  │
│  { name: "payment-service", ip: "10.0.1.42", port: 8080,    │
│    health: "/health", ttl: 30s }                             │
│                                                              │
│  Service A wants to call payment-service:                    │
│  → Queries registry: "where is payment-service?"            │
│  → Gets: ["10.0.1.42:8080", "10.0.1.43:8080"]              │
│  → Picks one (round robin / random)                          │
│  → Makes call directly                                       │
│                                                              │
│  Registry continuously polls /health on each instance.      │
│  If health check fails → deregisters instance.              │
│  Service A's next lookup won't include the dead instance.   │
└──────────────────────────────────────────────────────────────┘
```

### Client-Side vs Server-Side Discovery

```
CLIENT-SIDE DISCOVERY:
  Client queries registry → gets list of instances → picks one → calls directly
  (Netflix Eureka, Consul)
  
  Service A → Registry → [B1, B2, B3] → A picks B2 → A calls B2 directly
  
  ✅ Client controls load balancing algorithm
  ❌ Every client must implement discovery logic (each language/framework)

SERVER-SIDE DISCOVERY:
  Client calls load balancer → LB queries registry → LB forwards
  (AWS ALB, Kubernetes kube-proxy/CoreDNS)
  
  Service A → LB → (LB queries registry) → B2
  
  ✅ Client knows nothing about discovery — just calls a hostname
  ✅ One place for load balancing logic
  ❌ Extra hop (LB)
```

| Tool | Type | Notes |
|---|---|---|
| **Consul** | Self-hosted | Client + server side, health checks, KV store |
| **Eureka** | Self-hosted | Netflix OSS, Java-centric |
| **etcd** | Self-hosted | K8s native, key-value store for cluster state |
| **AWS Cloud Map** | Managed | DNS + API-based, integrated with ECS/EKS |
| **Kubernetes DNS** | Managed | CoreDNS — every service gets `service.namespace.svc.cluster.local` |
| **Zookeeper** | Self-hosted | Older, complex, used in Kafka/Hadoop ecosystems |

---

## MIGRATION PATTERNS

---

## 13. Strangler Fig

> **Problem:** You have a monolithic application built over years. You want to migrate to microservices. The "big bang rewrite" — stop everything, rewrite, launch — **fails more often than it succeeds** (famous failures: Netscape 6, healthcare.gov, many banking rewrites).

> **Solution:** Incrementally migrate functionality, one piece at a time. Route specific traffic to new microservices. The monolith shrinks. The microservice ecosystem grows. Eventually, the monolith is gone.

> **Origin:** Named after the strangler fig tree — a vine that grows around a tree, gradually envelops it, and eventually replaces it as the host tree dies.

```
┌──────────────────────────────────────────────────────────────────────┐
│                   STRANGLER FIG MIGRATION                            │
│                                                                      │
│  Phase 1: All traffic to monolith                                    │
│  Client → [Proxy/Gateway] → [     MONOLITH     ]                    │
│                                                                      │
│  Phase 2: Extract User Service                                       │
│  Client → [Proxy/Gateway] → /users/* → [User Service]  ← new       │
│                          → /orders/* → [   MONOLITH   ]             │
│                                                                      │
│  Phase 3: Extract Order Service                                      │
│  Client → [Proxy/Gateway] → /users/*  → [User Service]  ← new      │
│                          → /orders/* → [Order Service]  ← new      │
│                          → /other/*  → [   MONOLITH   ]            │
│                                                                      │
│  Phase N: Monolith replaced                                          │
│  Client → [API Gateway]  → /users/*    → [User Service]             │
│                          → /orders/*   → [Order Service]            │
│                          → /payments/* → [Payment Service]          │
│                                                                      │
│  Monolith: decommissioned ✅                                         │
└──────────────────────────────────────────────────────────────────────┘
```

### Implementation Steps

```
1. IDENTIFY a bounded context to extract (start small: read-only, low-risk)
2. BUILD the new microservice alongside the monolith
3. DEPLOY behind the API gateway/proxy
4. ROUTE traffic for that domain to the new service
5. VERIFY — metrics, errors, parity checks vs monolith
6. RETIRE the monolith code for that feature
7. REPEAT with the next domain
```

### Key Tactics

```
Feature Flags:
  Route 1% of /users traffic to new service
  Verify → increase to 10% → 50% → 100%
  Rollback instantly if issues arise

Anticorruption Layer (ACL):
  New service must talk to monolith's DB or API during transition.
  Wrap monolith interaction in an ACL — translate between models.
  Prevents new service from becoming coupled to monolith's data model.

Parallel Run:
  Send same request to both old and new service.
  Compare responses. Shadow traffic the new service.
  Only switch over when responses match.
```

| ✅ Pros | ❌ Cons |
|---|---|
| Zero big-bang risk | Long migration (months/years) |
| Rollback at any point | Running two systems simultaneously (cost) |
| Learnings from each phase inform the next | Proxy/gateway adds complexity |
| Business value delivered incrementally | Data migration is the hard part |

---

## 14. Shadow Deployment & Blue-Green

### Blue-Green Deployment

> Run two identical production environments. Switch all traffic from one to the other atomically. Instant rollback by switching back.

```
BEFORE DEPLOY:                          AFTER DEPLOY:
  Blue:  v1 ← 100% traffic               Blue:  v1 ← 0% traffic (keep alive for rollback)
  Green: v2 ← 0% (staging)               Green: v2 ← 100% traffic ✅

Switch: update DNS/LB rule → atomic.
Rollback: switch back to Blue instantly.

Cost: 2x infrastructure at all times.
Used by: AWS CodeDeploy, Spinnaker, Heroku
```

### Canary Deployment

```
Route small % of traffic to new version. Observe. Gradually increase.

v1: ████████████████████████████████████████████ 95%
v2: ████ 5%   ← canary — monitor error rate, latency

If v2 metrics good → increase to 20%, 50%, 100%
If v2 shows issues → route 0% to v2, rollback instantly
No downtime, minimal blast radius.
```

### Shadow / Traffic Mirroring

```
Copy real production traffic to a shadow environment.
Shadow responses are DISCARDED — not returned to user.
Used for: Load testing new service with real traffic, verifying parity.

Client → [LB] → Production Service → Response to client
              ↘ (mirrored copy) → Shadow Service → discarded
```

---

## 15. Pattern Decision Matrix

### Which Pattern for Which Problem?

| Problem | Pattern(s) |
|---|---|
| Cascading failure from slow dependency | Circuit Breaker |
| Transient network failures | Retry + Exponential Backoff + Jitter |
| One dependency starving resources from others | Bulkhead |
| Slow downstream hanging threads | Timeout |
| Service down but must partially function | Fallback + Graceful Degradation |
| Distributed transaction across multiple services | SAGA (Choreography or Orchestration) |
| DB write + event publish must be atomic | Transactional Outbox |
| High read/write asymmetry, different optimizations | CQRS |
| Full audit trail, time travel, replay | Event Sourcing |
| Cross-cutting concerns (TLS, retry, metrics) per service | Sidecar |
| Uniform policy across ALL services (mTLS, tracing) | Service Mesh |
| Services need to find each other dynamically | Service Discovery |
| Migrate monolith to microservices safely | Strangler Fig |
| Zero-downtime deploy with instant rollback | Blue-Green Deployment |
| Validate new version with real traffic safely | Canary Deployment |

### Resilience Pattern Stack (Use Together)

```
Recommended default resilience stack for any service-to-service call:

Timeout → Retry (backoff+jitter) → Circuit Breaker → Fallback

1. Timeout:         Don't wait forever
2. Retry:           Handle transient failures
3. Circuit Breaker: Stop calling a repeatedly-failing service
4. Fallback:        Return something useful when all else fails

Libraries: Resilience4j (Java), Polly (.NET), go-resiliency (Go), Envoy (mesh layer)
```

---

## 16. Interview Cheatsheet

### Quick Definitions

| Pattern | One-liner |
|---|---|
| **Circuit Breaker** | Trip after N failures, return fallback, probe for recovery |
| **Retry + Backoff** | Retry transient failures with exponential delay + jitter |
| **Bulkhead** | Isolate thread pools per dependency to contain failures |
| **Timeout** | Never wait indefinitely; set explicit deadlines at every layer |
| **Fallback** | Return cached/default data when primary path fails |
| **SAGA** | Distributed transaction via sequence of local txns + compensations |
| **Choreography SAGA** | Each service reacts to events — decentralized |
| **Orchestration SAGA** | Central orchestrator directs each step — centralized |
| **Transactional Outbox** | Write event to DB in same transaction; relay publishes it |
| **CQRS** | Separate write model (SQL) from read model (Redis/ES) |
| **Event Sourcing** | Store events, not state; current state = replay of events |
| **Sidecar** | Co-located proxy handles infra concerns (TLS, retry, tracing) |
| **Service Mesh** | Sidecars on every pod + control plane = network-wide policy |
| **Service Discovery** | Registry for dynamic lookup of healthy service instances |
| **Strangler Fig** | Incrementally replace monolith with microservices via routing |
| **Blue-Green** | Two envs; atomic traffic switch; instant rollback |
| **Canary** | Gradually shift traffic to new version; roll back on signals |

### When to Mention What in Interviews

| Interview Question | Patterns to Mention |
|---|---|
| "How would you handle failures in microservices?" | Circuit Breaker, Retry+Backoff, Bulkhead, Timeout, Fallback |
| "How would you handle distributed transactions?" | SAGA (choreography vs orchestration), Transactional Outbox |
| "Design a system with high read scalability" | CQRS (separate read/write models) |
| "How would you maintain full audit history?" | Event Sourcing |
| "How do microservices communicate reliably?" | Service Discovery, Service Mesh, Sidecar |
| "How would you migrate a monolith?" | Strangler Fig + Canary + Blue-Green |
| "How do you deploy with zero downtime?" | Blue-Green, Canary |
| "How do you prevent cascading failures?" | Circuit Breaker, Bulkhead, Timeout |
| "How do you publish events reliably?" | Transactional Outbox |
| "Service A calls Service B — what can go wrong?" | All 5 resilience patterns |

### Must-Know Interview Points

- ☑ **Circuit Breaker has 3 states:** CLOSED → OPEN → HALF-OPEN → CLOSED. Know the transitions.
- ☑ **Always add jitter to backoff** — prevents retry storms.
- ☑ **SAGA ≠ 2PC.** Sagas are eventually consistent, no locks, no blocking.
- ☑ **SAGA compensating transactions are NOT rollbacks** — they're new forward actions that undo the effect.
- ☑ **Choreography = decentralized, Orchestration = centralized.** Mix both in production.
- ☑ **Transactional Outbox** solves the dual-write problem — always mention with SAGA.
- ☑ **CQRS + Event Sourcing** are often used together but are independent patterns.
- ☑ **Sidecar → Service Mesh** is the progression from per-service to cluster-wide policy.
- ☑ **Strangler Fig** is the answer to "how do you migrate a monolith safely." Never big-bang.
- ☑ **Canary** for gradual rollout, **Blue-Green** for atomic switch. Know the difference.

---

*Sources: DesignGurus.io (19 Essential Microservices Patterns), AlgoMaster.io (Strangler Fig Pattern), ByteByteGo (SAGA Pattern Demystified), AWS Prescriptive Guidance (SAGA Choreography + Orchestration), Microsoft Azure Architecture Center (SAGA Pattern), Temporal.io (SAGA Mastery Guide), Baeldung CS (SAGA Pattern), Medium/Sylvain Tiset (Top 10 Microservices Patterns), ArXiv SLR (Resilient Microservices Recovery Patterns 2025), GeeksforGeeks (Microservices Design Patterns), DEV Community (SAGA Distributed Systems), Resilience4j Docs — combined with first-principles system design knowledge.*