# 🔭 Observability

> **Series:** System Design Notes  
> **Module:** 10 — Observability  
> **Prerequisites:** `08_api_design.md`, `11_system_design_patterns.md`, Basic microservices/Docker/Kubernetes concepts

---

## 📌 Table of Contents

1. [Observability vs Monitoring](#1-observability-vs-monitoring)
2. [The Three Pillars](#2-the-three-pillars)
3. [Pillar 1 — Logs](#3-pillar-1--logs)
4. [Pillar 2 — Metrics](#4-pillar-2--metrics)
5. [Pillar 3 — Distributed Tracing](#5-pillar-3--distributed-tracing)
6. [Correlating the Three Pillars](#6-correlating-the-three-pillars)
7. [OpenTelemetry — The Standard](#7-opentelemetry--the-standard)
8. [Alerting](#8-alerting)
9. [The SRE Golden Signals](#9-the-sre-golden-signals)
10. [SLI, SLO, SLA](#10-sli-slo-sla)
11. [Observability Stack & Tools](#11-observability-stack--tools)
12. [Real-World Architectures](#12-real-world-architectures)
13. [Common Mistakes](#13-common-mistakes)
14. [Interview Cheatsheet](#14-interview-cheatsheet)

---

## 1. Observability vs Monitoring

> These terms are often conflated. They are related but not the same.

### Monitoring — "Known Unknowns"

> Monitoring answers questions you already thought to ask. You define dashboards and alerts for problems you anticipate ahead of time.

```
"Is CPU > 80%?"      → You knew CPU could spike. You set a threshold.
"Is error rate > 1%?" → You knew errors could happen. You created the alert.

Monitoring is great for:
  ✅ Detecting expected failure modes
  ✅ Tracking known KPIs
  ✅ Infrastructure health checks

Monitoring fails when:
  ❌ A brand new failure mode occurs that you never anticipated
  ❌ You need to understand WHY something is broken, not just THAT it's broken
```

### Observability — "Unknown Unknowns"

> Observability is the ability to understand the **internal state of a system** from its **external outputs** — without having to add new instrumentation every time something breaks.

```
"Why is the checkout service slow for users in the Asia-Pacific region,
 but only when they have > 5 items in cart, and only during peak hours?"

→ You could NEVER have anticipated this exact combination.
→ An observable system lets you answer this ad-hoc from existing telemetry.
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                  MONITORING vs OBSERVABILITY                        │
│                                                                     │
│  Monitoring:                   Observability:                       │
│  ┌───────────────────────┐     ┌─────────────────────────────────┐  │
│  │ Pre-defined questions │     │ Arbitrary questions about       │  │
│  │ Dashboards + alerts   │     │ system state, answered from     │  │
│  │ you set up in advance │     │ existing telemetry              │  │
│  └───────────────────────┘     └─────────────────────────────────┘  │
│                                                                     │
│  Known Unknowns                Unknown Unknowns                     │
│  "We know errors can happen"   "We don't know what we don't know"   │
│                                                                     │
│  Monitoring IS a subset of observability.                           │
│  Observability ⊇ Monitoring.                                        │
└─────────────────────────────────────────────────────────────────────┘
```

> **The key test:** In an observable system, a developer can debug a novel production issue **entirely from telemetry data**, without adding new instrumentation or deploying new code.

---

## 2. The Three Pillars

> Observability is built on three types of telemetry data. Each answers a different question:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       THE THREE PILLARS                                  │
│                                                                          │
│  LOGS          METRICS         TRACES                                    │
│  ─────         ───────         ──────                                    │
│  "What         "How much /     "Where did                                │
│   happened?"    how fast?"      time go?"                                │
│                                                                          │
│  Discrete       Numeric         Request                                  │
│  events         aggregates      path across                              │
│  with context   over time       all services                             │
│                                                                          │
│  "Payment       "99th pct       "The request                             │
│   declined      latency =       spent 180ms                              │
│   for user 42"  450ms"          in DB query"                             │
└──────────────────────────────────────────────────────────────────────────┘
```

| Pillar | Answers | Granularity | Storage Cost | Best Tool |
|---|---|---|---|---|
| **Logs** | What happened and why | Per-event (high detail) | High | Elasticsearch, Loki |
| **Metrics** | How is the system performing | Aggregated numbers | Low | Prometheus, Datadog |
| **Traces** | Where did time go across services | Per-request path | Medium | Jaeger, Tempo, Zipkin |

---

## 3. Pillar 1 — Logs

> **Definition:** Logs are timestamped, immutable records of discrete events that occurred within a system. The oldest and most familiar observability signal.

### 3.1 Log Levels

```
TRACE   → Extremely verbose. Every function call. Dev only.
DEBUG   → Detailed internal state. Diagnosing specific issues. Dev/staging.
INFO    → Normal operational events. "User logged in", "Order placed". Production ✅
WARN    → Something unexpected happened but system is handling it.
ERROR   → An error occurred that affects a request/operation. Needs attention.
FATAL   → System is about to crash. Immediate action required.
```

```
Production guidance:
  Default level: INFO  (most services)
  During incident: DEBUG  (temporarily, then revert — cardinality explosion risk)
  Never in prod: TRACE  (performance impact + storage explosion)
```

---

### 3.2 Unstructured vs Structured Logs

```
UNSTRUCTURED (old way — avoid):
  2024-01-01 12:00:00 ERROR Payment failed for user 42 amount 99.99 card_id cc-123

  Problems:
  - Parsing with regex is fragile
  - Can't filter by field in log aggregation systems
  - Different formats across services = chaos

STRUCTURED (production standard — JSON):
  {
    "timestamp": "2024-01-01T12:00:00.000Z",
    "level": "ERROR",
    "service": "payment-service",
    "trace_id": "4bf92f3577b34da6",   ← links to distributed trace!
    "span_id": "00f067aa0ba902b7",
    "user_id": "user-42",
    "amount": 99.99,
    "card_id": "cc-123",
    "error": "card_declined",
    "message": "Payment processing failed",
    "env": "production",
    "region": "ap-south-1"
  }

  Benefits:
  ✅ Filter by any field: level=ERROR AND user_id=user-42
  ✅ Aggregate: COUNT(error=card_declined) GROUP BY region
  ✅ Correlate: click trace_id → see full distributed trace
  ✅ Machine-readable → plug into any log aggregation system
```

---

### 3.3 What to Log (and What Not To)

```
✅ ALWAYS LOG:
  - Service startup / shutdown
  - Every inbound and outbound request (at INFO level)
  - All errors and exceptions (with stack trace)
  - Authentication events (login, logout, token refresh)
  - Business-critical events (order placed, payment processed)
  - Configuration changes
  - External API calls (response time, status code)
  - Background job start/completion/failure

❌ NEVER LOG:
  - Passwords, tokens, secrets, API keys
  - Credit card numbers, CVV, full PAN
  - PII without masking (SSN, passport, health data — GDPR implications!)
  - Raw request bodies if they contain sensitive data
  - High-frequency operational noise (every cache hit/miss at DEBUG level)
```

---

### 3.4 Structured Logging in Code

```python
# Python — structlog
import structlog

log = structlog.get_logger()

def process_payment(user_id: str, amount: float, card_id: str):
    log = structlog.get_logger().bind(
        user_id=user_id,
        amount=amount,
        card_id=card_id[:4] + "****"   # mask card number
    )
    
    log.info("payment.started")
    
    try:
        result = payment_gateway.charge(card_id, amount)
        log.info("payment.success", transaction_id=result.id)
        return result
    except CardDeclinedError as e:
        log.warning("payment.declined", reason=str(e))
        raise
    except Exception as e:
        log.error("payment.error", error=str(e), exc_info=True)
        raise
```

```javascript
// Node.js — Winston
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()        // structured JSON output
  ),
  transports: [new winston.transports.Console()]
});

// Bind request context (trace_id from middleware)
const reqLogger = logger.child({
  trace_id: req.headers['x-trace-id'],
  user_id: req.user?.id,
  service: 'order-service'
});

reqLogger.info('order.created', { order_id: order.id, total: order.total });
reqLogger.error('order.failed', { error: err.message, stack: err.stack });
```

---

### 3.5 Log Pipeline Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        LOG PIPELINE                                    │
│                                                                        │
│  Application                                                           │
│  ┌───────────┐                                                         │
│  │ Service A │──► stdout/file                                          │
│  └───────────┘         │                                               │
│  ┌───────────┐         ▼                                               │
│  │ Service B │──► [ Log Shipper ] ← Fluentd / Filebeat / Vector       │
│  └───────────┘         │           (collect, parse, enrich, route)     │
│  ┌───────────┐         │                                               │
│  │ Service C │──►      │                                               │
│  └───────────┘         │                                               │
│                        ▼                                               │
│               [ Log Aggregation ]                                      │
│                        │                                               │
│          ┌─────────────┼───────────────┐                               │
│          ▼             ▼               ▼                               │
│  Elasticsearch    Loki (Grafana)   CloudWatch                          │
│  + Kibana         + Grafana        (AWS native)                        │
│  (ELK Stack)      (lightweight)                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**ELK Stack:**
- **Elasticsearch** — stores and indexes logs (inverted index → fast full-text search)
- **Logstash** / **Fluentd** / **Vector** — collect, parse, enrich, ship logs
- **Kibana** — visualize, query, dashboard, alert on logs

**Grafana Loki:**
- Cheaper than Elasticsearch — indexes only labels, not full text
- Queries logs by label (service, level, region) then filters within results
- Best for Kubernetes environments where labels are natural

---

## 4. Pillar 2 — Metrics

> **Definition:** Metrics are numeric measurements collected over time — aggregated snapshots of system behavior. Time-series data. Optimized for storage, querying, and alerting at scale.

> If logs tell you *what* happened, metrics tell you *how much*, *how fast*, and *how often*.

### 4.1 Metric Types

**Counter — monotonically increasing value**

```
Counts events. Only goes up. Reset to 0 on restart.

http_requests_total{method="GET", status="200"} = 148293
http_requests_total{method="POST", status="500"} = 42

Use: Total requests, total errors, total bytes sent
Query pattern: rate(http_requests_total[5m])  ← requests per second over 5m window
```

**Gauge — value that can go up or down**

```
Current snapshot. Replaces previous value.

memory_usage_bytes = 1073741824   (1 GB)
active_connections = 347
queue_depth = 1203

Use: CPU%, memory%, active connections, queue depth, temperature
Query pattern: memory_usage_bytes / node_memory_total_bytes * 100
```

**Histogram — distribution of values across buckets**

```
Counts observations falling into predefined size buckets.
Allows P50, P95, P99 latency calculations.

http_request_duration_seconds_bucket{le="0.1"}  = 24054  (requests < 100ms)
http_request_duration_seconds_bucket{le="0.5"}  = 33444  (requests < 500ms)
http_request_duration_seconds_bucket{le="1.0"}  = 34567  (requests < 1s)
http_request_duration_seconds_bucket{le="+Inf"} = 34567  (all requests)
http_request_duration_seconds_sum   = 15234.3   (total seconds spent)
http_request_duration_seconds_count = 34567     (total request count)

P99 = histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Summary — pre-calculated quantiles at client side**

```
Like histogram but quantiles calculated by the client (not server).
Less flexible — can't aggregate across instances.
Use histogram instead in most cases.
```

---

### 4.2 The Four Golden Signals (Google SRE)

> These four metrics are sufficient to understand the health of almost any service.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    FOUR GOLDEN SIGNALS                               │
│                                                                      │
│  1. LATENCY — how long does a request take?                          │
│     Measure: P50, P95, P99 (NOT average — averages hide outliers)   │
│     Alert on: P99 > threshold                                        │
│     Distinguish: successful vs error latency                         │
│                                                                      │
│  2. TRAFFIC — how much demand is the system handling?               │
│     Measure: requests/sec, queries/sec, messages/sec                 │
│     Use: capacity planning, detect traffic spikes                    │
│                                                                      │
│  3. ERRORS — what fraction of requests are failing?                  │
│     Measure: error rate = errors / total requests                    │
│     Include: explicit 5xx, implicit (200 with wrong data)            │
│     Alert on: error_rate > 0.1%                                      │
│                                                                      │
│  4. SATURATION — how "full" is the service?                          │
│     Measure: CPU%, memory%, connection pool%, queue depth            │
│     Leading indicator: high saturation → latency will increase soon  │
│     Alert on: CPU > 80% sustained, queue depth growing               │
└──────────────────────────────────────────────────────────────────────┘
```

---

### 4.3 Why Percentiles, Not Averages

```
Example: 100 requests, 99 take 10ms, 1 takes 10,000ms (10 seconds)

Average: (99 × 10 + 10000) / 100 = 109.9ms  ← "looks fine"
P99:     10,000ms                            ← "1 in 100 users waits 10 seconds" 🔴

Average HIDES tail latency. Users experiencing the 99th percentile are your
most valuable/heavy users — they make the most requests.

Always track:
  P50  (median) — the typical user experience
  P95           — 1 in 20 users experience this
  P99           — 1 in 100 users experience this
  P999          — 1 in 1000 users (for critical systems: payments, safety)
```

---

### 4.4 Prometheus + Grafana Stack

**Prometheus:**
- Pull-based metrics collection (scrapes `/metrics` endpoint on each service)
- Time-series database optimized for metrics
- PromQL query language
- Built-in alerting (Alertmanager)

```yaml
# prometheus.yml — scrape config
scrape_configs:
  - job_name: 'payment-service'
    static_configs:
      - targets: ['payment-service:8080']
    metrics_path: '/metrics'
    scrape_interval: 15s

  - job_name: 'kubernetes-pods'    # autodiscovery in k8s
    kubernetes_sd_configs:
      - role: pod
```

```python
# Python — Prometheus client instrumentation
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'Request latency',
    ['endpoint'],
    buckets=[.01, .05, .1, .25, .5, 1, 2.5, 5, 10]
)
ACTIVE_CONNECTIONS = Gauge('active_connections', 'Current active connections')

@app.route('/pay', methods=['POST'])
def pay():
    ACTIVE_CONNECTIONS.inc()
    with REQUEST_LATENCY.labels(endpoint='/pay').time():
        try:
            result = process_payment()
            REQUEST_COUNT.labels('POST', '/pay', '200').inc()
            return result
        except Exception as e:
            REQUEST_COUNT.labels('POST', '/pay', '500').inc()
            raise
        finally:
            ACTIVE_CONNECTIONS.dec()
```

**PromQL query examples:**

```promql
# Request rate (per second, 5-minute window)
rate(http_requests_total[5m])

# Error rate percentage
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m]) * 100

# P99 latency
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m]))

# Memory usage percentage
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# CPU usage per core
100 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
```

---

## 5. Pillar 3 — Distributed Tracing

> **Definition:** Distributed tracing records the complete journey of a single request as it propagates through multiple services. Each hop is recorded as a **span**. All spans for one request form a **trace**.

> Without tracing, a slow request in a microservices system requires logging into 10 different services, searching 10 different log streams, manually correlating timestamps. Tracing collapses this to a single waterfall view.

### 5.1 Core Concepts

```
TRACE:
  The complete end-to-end record of one request flowing through your system.
  Identified by a unique Trace ID (128-bit hex string).

SPAN:
  One unit of work within a trace. Represents a single operation.
  Has: name, start time, duration, status, tags/attributes.
  Spans are nested in a parent-child tree.

TRACE ID: 4bf92f3577b34da6a3ce929d0e0e4736  ← shared across all services
SPAN ID:  00f067aa0ba902b7                   ← unique to each operation
PARENT SPAN ID: 0000000000000000             ← root span has no parent
```

### 5.2 How a Trace Looks

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
Total duration: 340ms

Waterfall diagram:
┌──────────────────────────────────────────────────────────────────────┐
│ Span: API Gateway (root)                          [0ms ─────── 340ms]│
│   Span: Auth Middleware                           [5ms ── 15ms]      │
│   Span: Order Service                             [20ms ─────── 310ms]
│     Span: Validate Request                        [20ms ─ 25ms]      │
│     Span: DB Query (SELECT order)                 [30ms ─── 90ms] ← SLOW
│     Span: Payment Service gRPC call               [95ms ─── 250ms]   │
│       Span: Fraud Check (internal)                [100ms ─ 120ms]    │
│       Span: Card Processor API                    [125ms ─── 240ms]← SLOW
│     Span: Inventory Service                       [255ms ─ 275ms]    │
│     Span: DB Write (INSERT order)                 [280ms ─ 305ms]    │
│   Span: Response serialization                    [310ms ─ 315ms]    │
└──────────────────────────────────────────────────────────────────────┘

Root cause identified: Card Processor API (external) taking 115ms
→ Either cache, timeout aggressively, or switch processors
```

---

### 5.3 Context Propagation

> Trace context (trace_id, span_id) must be **propagated in headers** across every service boundary. Without this, you get disconnected spans instead of a unified trace.

```
W3C TraceContext header (OpenTelemetry standard):
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
               │  │                                │                │
               │  └─ Trace ID (128-bit)            │                └─ sampled
               └─ version                          └─ Span ID (64-bit)

B3 header (older, Zipkin/Jaeger):
  X-B3-TraceId: 4bf92f3577b34da6a3ce929d0e0e4736
  X-B3-SpanId: 00f067aa0ba902b7
  X-B3-Sampled: 1
```

```python
# Automatic context propagation (OpenTelemetry — no manual header work)
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor

FlaskInstrumentor().instrument()     # auto-instruments all Flask routes
RequestsInstrumentor().instrument()  # auto-propagates trace context in all HTTP calls

# Manual span creation (for business logic)
from opentelemetry import trace

tracer = trace.get_tracer("payment-service")

def process_payment(order_id: str, amount: float):
    with tracer.start_as_current_span("payment.process") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("payment.amount", amount)
        
        with tracer.start_as_current_span("fraud.check") as fraud_span:
            result = fraud_service.check(order_id)
            fraud_span.set_attribute("fraud.score", result.score)
        
        with tracer.start_as_current_span("card.charge"):
            return card_processor.charge(amount)
```

---

### 5.4 Sampling Strategies

> Tracing every request at 100% is too expensive at scale. Sampling reduces volume while keeping useful data.

```
HEAD-BASED SAMPLING (decision at request start):
  Sample 10% of all requests uniformly.
  Simple, low overhead.
  Problem: You might miss the 1 slow request you care about.

TAIL-BASED SAMPLING (decision after trace is complete):
  See the full trace → then decide whether to keep it.
  ✅ Always keep ERROR traces
  ✅ Always keep SLOW traces (>1s)
  ✅ Keep 10% of everything else

  Cost: Must buffer traces until complete (memory overhead).
  Worth it: you keep ALL the interesting traces.

EXEMPLARS (metric → trace link):
  When a histogram bucket fires, attach the trace_id of the request that caused it.
  Jump from "P99 latency spike at 14:32" → exact trace ID → see what was slow.
```

```yaml
# OpenTelemetry Collector — tail-based sampling config
processors:
  tail_sampling:
    decision_wait: 10s   # buffer traces for 10s before deciding
    policies:
      - name: keep-errors
        type: status_code
        status_code: { status_codes: [ERROR] }        # 100% of errors

      - name: keep-slow
        type: latency
        latency: { threshold_ms: 1000 }               # 100% of slow traces

      - name: sample-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }    # 10% of normal traces
```

---

## 6. Correlating the Three Pillars

> The real power of observability is when the three pillars work **together** — each signal enriches the others. Correlation turns three data silos into a unified debugging workflow.

### The Golden Path: Alert → Metric → Trace → Log

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   DEBUGGING WORKFLOW (correlation in action)            │
│                                                                         │
│  Step 1: ALERT fires                                                    │
│    "P99 latency on /checkout > 2s for 5 minutes"  (METRIC alert)       │
│                                                                         │
│  Step 2: Open METRIC dashboard                                          │
│    → Confirm spike started at 14:32                                     │
│    → Drill down by service: Payment Service shows high latency          │
│    → Exemplar on the histogram: trace_id = "4bf92f3577b34da6"          │
│                                                                         │
│  Step 3: Open TRACE                                                     │
│    → Waterfall shows Card Processor API call taking 1.8s               │
│    → span_id = "00f067aa0ba902b7"                                       │
│                                                                         │
│  Step 4: Open LOGS (filtered by trace_id)                               │
│    → trace_id="4bf92f3577b34da6" AND service="payment-service"          │
│    → Log: "Card processor timeout after 1800ms, retrying..."           │
│    → Log: "Retry 1 also timed out"                                      │
│    → Log: "Circuit breaker opened for card-processor"                   │
│                                                                         │
│  Root cause found in < 5 minutes: Card processor is degraded.          │
│  Action: Switch to backup processor, notify card processor SRE.         │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Linchpin: `trace_id` in Every Log

```
THIS IS THE MOST IMPORTANT CORRELATION MECHANISM.
Every single log line must contain the trace_id of the current request.

Without trace_id in logs:
  You have a trace showing Payment Service is slow.
  You go to logs and search: service=payment-service level=ERROR
  You get 10,000 error logs. Which one is YOUR request? 🤔

With trace_id in logs:
  trace_id="4bf92f3577b34da6" service=payment-service
  → 3 log lines. Exactly the ones for your request. ✅
```

```python
# FastAPI/Python — inject trace_id into every log line
from opentelemetry import trace
import structlog

def get_trace_id() -> str:
    span = trace.get_current_span()
    ctx = span.get_span_context()
    return format(ctx.trace_id, '032x') if ctx.is_valid else ""

# Bind trace_id to logger context — propagates to all subsequent logs
log = structlog.get_logger().bind(trace_id=get_trace_id())
```

---

## 7. OpenTelemetry — The Standard

> **OpenTelemetry (OTel)** is a vendor-neutral, open-source observability framework. It provides a single SDK to instrument your code once, and send telemetry to ANY backend (Datadog, Grafana, Jaeger, New Relic, etc.) without vendor lock-in.

```
BEFORE OpenTelemetry (the problem):
  Use Datadog → instrument with Datadog SDK
  Switch to Grafana → rip out Datadog SDK, add Grafana SDK
  Add New Relic for a team → another SDK, another format

WITH OpenTelemetry:
  Instrument ONCE with OTel SDK
  → OTel Collector routes to ANY backend
  → Switch backends without code changes ✅
```

### OpenTelemetry Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                      OPENTELEMETRY ARCHITECTURE                       │
│                                                                       │
│  Application Code                                                     │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ OTel SDK  (auto-instrument HTTP, DB, gRPC + manual spans)    │    │
│  │ Generates: Traces + Metrics + Logs  in OTLP format           │    │
│  └─────────────────────────┬────────────────────────────────────┘    │
│                             │ OTLP (gRPC or HTTP)                    │
│                             ▼                                         │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │              OTel COLLECTOR                                  │    │
│  │  Receives → Processes (sample, enrich, filter) → Exports    │    │
│  └─────────────────────────┬────────────────────────────────────┘    │
│                             │                                         │
│           ┌─────────────────┼────────────────────┐                   │
│           ▼                 ▼                    ▼                   │
│     Grafana Tempo      Prometheus          Datadog / New Relic       │
│     (traces)           (metrics)           (commercial all-in-one)   │
│           ▼                                                           │
│     Grafana Loki                                                      │
│     (logs)                                                            │
└───────────────────────────────────────────────────────────────────────┘
```

### Auto-Instrumentation (Zero Code Changes)

```bash
# Python — instrument an existing Flask app with zero code changes
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install   # auto-detects installed libraries

# Run with auto-instrumentation:
OTEL_SERVICE_NAME=payment-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317 \
opentelemetry-instrument python app.py

# Auto-instruments: Flask routes, SQLAlchemy queries, Redis calls,
#                   requests library calls, logging
```

```bash
# Java — zero-code agent instrumentation
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -jar order-service.jar
```

---

## 8. Alerting

> An alert is a signal that something requires human attention. The goal: **alert on symptoms, not causes.**

### Alert on Symptoms, Not Causes

```
CAUSE-BASED (bad):
  "Redis CPU > 80%"
  → Redis might be at 80% and everything is fine.
  → Or Redis is at 80% and serving users perfectly.
  → This alert may never matter to end users.

SYMPTOM-BASED (good):
  "P99 checkout latency > 2s for 5 minutes"
  → Users are experiencing slow checkouts RIGHT NOW.
  → This definitely matters.
  → You then investigate the cause (which might be Redis, or something else).

Rule: Alert on what users experience. Investigate the root cause after.
```

---

### Alert Quality — Avoiding Alert Fatigue

```
BAD ALERTS (cause alert fatigue → engineers ignore all alerts):
  ❌ Too sensitive: fires every time CPU spikes for 30 seconds
  ❌ Too noisy: fires 50 times during one incident
  ❌ Not actionable: "Memory increased" — what do I do with this?
  ❌ No runbook: engineer receives alert with no guidance

GOOD ALERTS:
  ✅ Sustained: condition must hold for 5+ minutes (reduces flapping)
  ✅ Deduplicated: one alert per incident, not one per symptom
  ✅ Actionable: "Checkout P99 > 2s — see runbook: runbooks.company.com/checkout-latency"
  ✅ Severity tiered: P1 (page immediately) vs P2 (notify Slack) vs P3 (ticket)
```

### Alert Routing

```
Alert fires
    ↓
Alertmanager / PagerDuty
    │
    ├── P1 (SEV-1): Pages on-call engineer (phone call) + Slack #incidents
    │   Examples: Error rate > 10%, Service down, Data loss
    │
    ├── P2 (SEV-2): Slack #alerts, creates ticket, no page
    │   Examples: P99 > 2s, Error rate 1-10%, Queue lag growing
    │
    └── P3 (SEV-3): Creates ticket only
        Examples: Disk > 80%, Memory > 85%, Non-critical job failures
```

---

### Prometheus Alerting Rules

```yaml
# payment-alerts.yaml
groups:
  - name: payment-service
    rules:
      # High error rate — symptom-based alert
      - alert: PaymentHighErrorRate
        expr: |
          rate(http_requests_total{service="payment",status=~"5.."}[5m])
          / rate(http_requests_total{service="payment"}[5m]) > 0.01
        for: 5m     # must be true for 5 minutes (reduce flapping)
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Payment service error rate > 1%"
          description: "Current error rate: {{ $value | humanizePercentage }}"
          runbook: "https://runbooks.company.com/payment-errors"
          dashboard: "https://grafana.company.com/d/payments"

      # P99 latency — symptom-based
      - alert: PaymentHighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket{service="payment"}[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Payment P99 latency > 2s"
```

---

## 9. The SRE Golden Signals

> From Google's SRE Book — the four metrics every service must expose and monitor.

| Signal | Measure | Alert Condition |
|---|---|---|
| **Latency** | P50, P95, P99 response time | P99 > SLO threshold for 5 min |
| **Traffic** | Requests/sec, queries/sec | Sudden spike or drop (anomaly) |
| **Errors** | Error rate (5xx / total) | Error rate > 1% for 5 min |
| **Saturation** | CPU%, memory%, queue depth | CPU > 80% sustained, queue growing |

> **Saturation is a leading indicator.** High saturation predicts future latency spikes before users notice them.

### RED Method (for Services)

> Simpler alternative to Golden Signals — specifically for request-driven services:

| Letter | Signal | Example |
|---|---|---|
| **R** — Rate | Requests per second | `rate(http_requests_total[5m])` |
| **E** — Errors | Error rate | `rate(http_errors_total[5m])` |
| **D** — Duration | Latency distribution | `histogram_quantile(0.99, ...)` |

### USE Method (for Resources)

> For infrastructure/resource monitoring (CPU, memory, disk, network):

| Letter | Signal | Example |
|---|---|---|
| **U** — Utilization | % time resource is busy | CPU 73% |
| **S** — Saturation | How much extra work is queued | Run queue length = 5 |
| **E** — Errors | Error count | Disk write errors = 12 |

---

## 10. SLI, SLO, SLA

> The language of reliability targets. Used in SRE teams to define and measure service quality.

### Definitions

```
SLI — Service Level Indicator
  A metric that measures service quality as experienced by users.
  "The thing you actually measure"
  
  Examples:
    Availability SLI:  successful_requests / total_requests
    Latency SLI:       % of requests completing in < 200ms
    Error SLI:         1 - (error_count / total_count)

SLO — Service Level Objective
  The TARGET value for an SLI. Internal commitment.
  "The threshold you aim to stay above"
  
  Examples:
    "99.9% of requests return a successful response" (availability)
    "95% of requests complete in < 200ms" (latency)
    "Error rate < 0.1% over a 30-day rolling window"

SLA — Service Level Agreement
  A CONTRACTUAL commitment to customers. Violation = financial penalty.
  "The legal contract with customers"
  Always looser than your internal SLO (buffer for unexpected failures).
  
  Examples:
    "We guarantee 99.9% uptime per month. Breach = credits."
    SLO might be 99.95% internally, giving buffer before breaching SLA.
```

### Error Budget

```
SLO: 99.9% availability over 30 days

Total minutes in 30 days: 43,200 minutes
Error budget:  0.1% × 43,200 = 43.2 minutes of allowed downtime

Error budget usage:
  Week 1: Deploy incident → 10 min downtime. Budget used: 10/43.2 = 23%
  Week 2: DB failover    →  5 min downtime. Budget used: 15/43.2 = 35%
  Week 3: No incidents.  Budget used: 15/43.2 = 35%
  Week 4: Bad deploy     → 30 min downtime. Budget used: 45/43.2 = 104% 🔴
          → SLO BREACHED. Feature freeze. Focus on reliability.

Error budget forces the balance between:
  Moving fast (deploys, experiments) → consumes error budget
  Staying reliable                   → preserves error budget
```

---

## 11. Observability Stack & Tools

### Open Source Stack (Grafana Stack)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GRAFANA STACK                                │
│                                                                     │
│  Metrics: Prometheus → Grafana                                      │
│  Logs:    Promtail/Fluentd → Loki → Grafana                         │
│  Traces:  OTel SDK → OTel Collector → Tempo → Grafana               │
│  Alerts:  Prometheus Alertmanager → PagerDuty / Slack               │
│                                                                     │
│  All visualized in Grafana — single pane of glass.                  │
│  Click from metric → drill to trace → click to correlated logs.    │
└─────────────────────────────────────────────────────────────────────┘
```

### Commercial All-in-One

| Tool | Strengths | Pricing |
|---|---|---|
| **Datadog** | Best-in-class UX, APM, AI insights, 500+ integrations | Expensive (per host/GB) |
| **New Relic** | Full stack, good ML anomaly detection | Per-user or per-GB |
| **Dynatrace** | AI-powered root cause, PurePath tracing | Enterprise pricing |
| **Grafana Cloud** | Open-source tools managed, cheaper than above | Free tier available |
| **Honeycomb** | Column-store optimized for high-cardinality tracing | Per event |

### AWS Native Stack

| Signal | Service |
|---|---|
| **Logs** | CloudWatch Logs + Log Insights |
| **Metrics** | CloudWatch Metrics + Embedded Metrics Format |
| **Traces** | AWS X-Ray |
| **Dashboards** | CloudWatch Dashboards |
| **Alerts** | CloudWatch Alarms → SNS → PagerDuty |
| **Collector** | AWS Distro for OpenTelemetry (ADOT) |

### Tool Comparison

| Use Case | Open Source | AWS Native | Commercial |
|---|---|---|---|
| Metrics | Prometheus | CloudWatch | Datadog |
| Logs | Loki / ELK | CloudWatch Logs | Datadog / Splunk |
| Traces | Jaeger / Tempo | X-Ray | Datadog APM / Dynatrace |
| Dashboards | Grafana | CloudWatch | Datadog |
| Alerting | Alertmanager | CloudWatch Alarms | PagerDuty |

---

## 12. Real-World Architectures

### Netflix — ATOM (Atlas + Edgar + Mantis)

```
Metrics: Atlas (in-house time-series DB)
  - Handles millions of metrics per second
  - Tag-based query model
  - Spans all AWS regions globally

Traces: Edgar
  - Custom distributed tracing built on Zipkin concepts
  - Every streaming request traced end-to-end

Stream processing: Mantis
  - Real-time anomaly detection on the metrics stream
  - Fires alerts before Prometheus-style polling would catch them

Scale: 260M subscribers → billions of playback events per day
```

### Uber — M3 + Jaeger

```
Metrics: M3 (open-sourced, Prometheus-compatible)
  - Built because Prometheus couldn't handle Uber's cardinality
  - Millions of unique time series
  - Distributed storage layer (M3DB)

Traces: Jaeger (also open-sourced by Uber)
  - 10+ billion spans per day
  - Adaptive sampling (rate per operation, per service)
  - Used by hundreds of companies globally (CNCF project)
```

### Typical Startup Observability Stack

```
Phase 1 (early, <10 services):
  Logs:    CloudWatch Logs (free with AWS)
  Metrics: CloudWatch Metrics (basic)
  Alerts:  CloudWatch Alarms → Slack
  Traces:  None yet (add when debugging becomes painful)

Phase 2 (growing, 10–50 services):
  Logs:    ELK Stack or Grafana Loki
  Metrics: Prometheus + Grafana
  Traces:  Jaeger or Tempo + OTel instrumentation
  Alerts:  Alertmanager → PagerDuty

Phase 3 (scale, 50+ services):
  Full OTel SDK across all services
  Grafana unified stack OR Datadog
  Tail-based sampling for traces
  SLO dashboards + error budget tracking
  Dedicated SRE team
```

---

## 13. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **No `trace_id` in logs** | Can't correlate logs to a specific request/trace | Inject trace_id into every log via middleware |
| **Logging passwords, tokens, PII** | Security breach, GDPR violation | Sanitize all log fields before writing |
| **Alert on causes, not symptoms** | Alert fatigue, engineers ignore alerts | Alert on P99 latency / error rate (user-facing) |
| **Alerting on averages** | Averages hide tail latency — 1% of users silently suffering | Always alert on P99 percentiles |
| **100% trace sampling** | Storage explosion at scale, performance overhead | Tail-based: 100% errors + slow, 10% rest |
| **No runbook linked to alert** | On-call engineer gets paged at 3am with no guidance | Every alert has a `runbook` URL annotation |
| **Separate tools for logs/metrics/traces** | Can't correlate — switching between 3 UIs in an incident | Unified platform or Grafana linking all three |
| **Only metric monitoring (no traces)** | "Payment service is slow" but can't tell which DB query | Distributed tracing shows exactly where time is spent |
| **SLO set too high** | Zero error budget — no feature releases allowed | Start at 99.9% (43 min/month), not 99.999% |
| **No error budget process** | SLO is a vanity number nobody acts on | Link error budget to engineering policy (feature freeze when exhausted) |
| **Instrumentation only at entry point** | Internal bottlenecks invisible | Instrument at every service boundary (in + out) |

---

## 14. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Observability** | Ability to understand internal system state from external outputs |
| **Monitoring** | Pre-defined checks for known failure modes (subset of observability) |
| **Log** | Timestamped, immutable record of a discrete event |
| **Metric** | Numeric time-series measurement (counter, gauge, histogram) |
| **Trace** | Full journey of one request across all services |
| **Span** | One unit of work within a trace |
| **Trace ID** | Unique ID linking all spans for one request across all services |
| **P99** | 99th percentile latency — 1 in 100 requests experience this or worse |
| **SLI** | The metric you measure to assess service quality |
| **SLO** | Your internal target for the SLI |
| **SLA** | Contractual commitment to customers — always looser than SLO |
| **Error Budget** | 1 - SLO = allowed downtime/error rate before breach |
| **Golden Signals** | Latency, Traffic, Errors, Saturation — four metrics for any service |
| **OpenTelemetry** | Vendor-neutral SDK + standard for emitting all three telemetry types |
| **Tail-based sampling** | Sample decision made after seeing complete trace — keeps errors/slow traces |

### When to Use What (Scenarios)

| Scenario | Tool / Approach |
|---|---|
| "Service is down" investigation | Metrics (error rate) → Traces (which service) → Logs (why) |
| Debug slow specific request | Trace waterfall — shows every span + latency breakdown |
| Capacity planning | Metrics (traffic + saturation trends over time) |
| Find all logs for one user request | Filter logs by `trace_id` |
| Alert on service degradation | Prometheus alert on P99 latency + error rate (symptom-based) |
| Track business metrics | Custom counter metrics (orders/sec, revenue/sec) |
| Understand internal service CPU | USE method (Utilization, Saturation, Errors) |
| Debug microservice request flows | Distributed tracing (Jaeger / Tempo) |
| Set reliability targets | SLI → SLO → SLA + error budget |
| Vendor-neutral instrumentation | OpenTelemetry SDK + Collector |

### The Interview Answer Template

```
When asked "How would you make system X observable?":

1. LOGS:
   "Structured JSON logs with trace_id on every line.
    Shipped to Elasticsearch/Loki via Fluentd.
    Visualized in Grafana/Kibana."

2. METRICS (Four Golden Signals):
   "Expose Prometheus metrics: request rate, P99 latency, error rate, saturation.
    Alerts on P99 > 2s and error rate > 1% — symptom-based, 5-minute window."

3. TRACES:
   "OpenTelemetry SDK on every service. Tail-based sampling —
    100% of errors and slow traces, 10% of normal.
    Stored in Grafana Tempo or Jaeger."

4. CORRELATION:
   "trace_id injected into every log line.
    Grafana exemplars link metric spikes to specific traces.
    Single pane of glass: alert → metric → trace → log."

5. SLOs:
   "Define SLI (% requests < 200ms) and SLO (99.9%).
    Error budget tracks deployment cadence vs reliability.
    Feature freeze when error budget is exhausted."
```

### Must-Know Interview Points

- ☑ **Observability ⊇ Monitoring.** Monitoring = known unknowns. Observability = unknown unknowns too.
- ☑ **Three pillars:** Logs (what), Metrics (how much), Traces (where time went).
- ☑ **trace_id in every log line** is the most important correlation mechanism.
- ☑ **Alert on symptoms** (P99 latency, error rate) not causes (CPU, Redis memory).
- ☑ **P99 not average** — averages hide tail latency, percentiles expose it.
- ☑ **Tail-based sampling** = sample AFTER seeing full trace → keep all interesting traces.
- ☑ **SLO < SLA** always — internal target must be tighter than contractual commitment.
- ☑ **Error budget = 1 - SLO** — spend it on features, preserve it for reliability.
- ☑ **OpenTelemetry** = instrument once, send anywhere — the answer to vendor lock-in.
- ☑ **Structured logs** (JSON) are mandatory — unstructured text is unqueryable at scale.

---

*Sources: OpenTelemetry Docs (Observability Primer), IBM Think (Three Pillars of Observability), Elastic Blog (Pillars + Profiling), BackendBytes (Discipline of Observability — Feb 2026), SigNoz Blog (Three Pillars), Netdata Academy (Observability Pillars), Datadog Knowledge Center, Spacelift Blog (OTel Implementation), OpenObserve Blog (Microservices Observability) — combined with first-principles system design and SRE knowledge.*