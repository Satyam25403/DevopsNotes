# OpenTelemetry - Core Concepts: Spans, Traces & Context Propagation

The core theory behind how OpenTelemetry represents a request's journey across multiple services.

## Table of Contents
- [Spans Deep Dive](#spans-deep-dive)
- [Traces: Spans Connected Together](#traces-spans-connected-together)
- [Span Attributes and Events](#span-attributes-and-events)
- [Context Propagation Explained](#context-propagation-explained)
- [Resources: Identifying the Source](#resources-identifying-the-source)
- [Semantic Conventions](#semantic-conventions)
- [Metrics and Logs Signal Types](#metrics-and-logs-signal-types)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Spans Deep Dive

**A Span represents ONE unit of work — a single operation with a start time, end time, and outcome.**

**Visual:**
```
A Single Span:
┌─────────────────────────────────────┐
│  Name: "SELECT orders WHERE user_id=?"     │
│  Start: 10:15:32.100                            │
│  End:   10:15:32.145                              │
│  Duration: 45ms                                     │
│  Status: OK                                            │
│  Attributes:                                                │
│    db.system: postgresql                                       │
│    db.statement: "SELECT * FROM orders..."                        │
└─────────────────────────────────────┘

Every span has:
- A unique Span ID
- A Parent Span ID (unless it's the root span)
- A start/end timestamp
- A status (OK, ERROR, UNSET)
- Attributes (key-value metadata)
```

---

## Traces: Spans Connected Together

**A Trace is the FULL collection of spans representing one complete request, connected by a shared Trace ID.**

**Visual:**
```
Trace ID: a1b2c3d4 (shared across ALL spans in this request)

┌─────────────────────────────────────────────┐
│  Span: "POST /checkout" (root span, no parent)     │
│  ├── Span: "validate-cart" (parent: checkout)         │
│  ├── Span: "charge-payment" (parent: checkout)           │
│  │     └── Span: "stripe-api-call" (parent: charge)         │
│  └── Span: "send-confirmation-email" (parent: checkout)         │
└─────────────────────────────────────────────┘

Visualized as a waterfall (what Jaeger shows):
POST /checkout          ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ 320ms
├─validate-cart              ▓▓ 15ms
├─charge-payment                  ▓▓▓▓▓▓▓▓▓▓▓▓▓ 250ms
│  └─stripe-api-call                  ▓▓▓▓▓▓▓▓▓▓▓ 230ms
└─send-email                                      ▓▓ 20ms
```

**Visual — the parent-child relationship is what creates the tree:**
```
Each span knows its PARENT span ID.
The TRACE is simply: "all spans sharing the same
Trace ID, organized into a tree by their parent-child
relationships." This is a simple, powerful data model —
no separate "trace" object needs to be explicitly built,
it emerges naturally from spans referencing their parents.
```

---

## Span Attributes and Events

**Attributes add searchable, structured metadata to a span. Events mark a specific MOMENT within a span's duration.**

```python
with tracer.start_as_current_span("charge-payment") as span:
    span.set_attribute("payment.method", "credit_card")
    span.set_attribute("payment.amount", 49.99)

    span.add_event("payment_gateway_called")
    # ... call payment gateway ...
    span.add_event("payment_gateway_responded")

    if payment_failed:
        span.set_status(Status(StatusCode.ERROR, "Card declined"))
```

**Visual:**
```
Attributes vs Events:
┌───────────────────────────────────────────┐
│  Attribute        → Describes the WHOLE span       │
│                   (payment.method = "credit_card")     │
│                                                  │
│  Event                → Marks a MOMENT within the span      │
│                   ("payment_gateway_called" at 10:15:32.110)│
└───────────────────────────────────────────┘

Why this matters for debugging:
Attributes let you FILTER/search traces (e.g. "show me
all traces where payment.method = credit_card AND
status = ERROR"). Events show the internal timeline
of what happened DURING a single span, useful for
spans that do several sequential things internally.
```

---

## Context Propagation Explained

**This is the mechanism that lets a trace span MULTIPLE SERVICES, not just one process.**

**Visual:**
```
Without Context Propagation:
Service A generates a trace → calls Service B via HTTP →
   Service B has NO IDEA it's part of an ongoing trace →
   Service B starts its OWN, completely disconnected trace

Result: TWO separate, unrelated traces instead of ONE
        unified view of the whole request's journey.

With Context Propagation:
Service A generates trace ID "a1b2c3d4" →
   makes an HTTP call to Service B, INJECTING the
   trace context into HTTP headers:
   traceparent: 00-a1b2c3d4...-00f067aa...-01
        ↓
   Service B's OTel SDK EXTRACTS this header →
   continues the SAME trace, creating child spans
   with the SAME trace ID "a1b2c3d4"

Result: ONE unified trace spanning BOTH services,
        exactly like the checkout example above.
```

**Visual — the W3C Trace Context standard header:**
```
traceparent: 00-a1b2c3d4e5f6...-00f067aa0ba902b7-01
             │  │                    │                │
           version  trace-id      parent-span-id   flags

This is a W3C STANDARD (not OTel-specific) — meaning
even services instrumented with DIFFERENT tools, as
long as they respect this standard header, can
participate in the same distributed trace.
```

**Why auto-instrumentation matters here:**
```
Manually propagating context (reading/injecting headers
on every single HTTP call) would be extremely tedious
and error-prone to do by hand across dozens of services.

Auto-instrumentation libraries (for Flask, Express,
requests, axios, etc.) handle this AUTOMATICALLY —
injecting/extracting the traceparent header on every
outgoing/incoming request, without any manual code.
```

---

## Resources: Identifying the Source

**A Resource describes WHERE telemetry came from — attached once, applies to everything that service emits.**

```python
from opentelemetry.sdk.resources import Resource

resource = Resource(attributes={
    "service.name": "checkout-service",
    "service.version": "2.3.1",
    "deployment.environment": "production",
    "host.name": "checkout-7f8d9c-x2p1",
})
```

**Visual:**
```
Why Resources matter:
Without them, all your spans/metrics look identical
regardless of WHICH service instance produced them —
impossible to filter "show me only traces from
checkout-service version 2.3.1 in production."

With Resources attached:
Every span/metric/log automatically carries this
identifying metadata, letting you slice/filter/group
telemetry by service, version, environment, or host
in your backend (Jaeger, Grafana, etc.) — exactly
the same filtering concepts covered in the Grafana
and Loki notes (labels for logs, dimensions for metrics).
```

---

## Semantic Conventions

**OpenTelemetry defines STANDARD attribute names for common concepts — so a "database call" span looks the same regardless of which language/library generated it.**

**Visual:**
```
Without semantic conventions:
Team A's spans:  db_type: "postgres", query_text: "..."
Team B's spans:    database.kind: "postgresql", sql: "..."
Team C's spans:      dbSystem: "postgres", statement: "..."
→ THREE different attribute names for the SAME concept,
  impossible to write ONE consistent dashboard/query
  across all three teams' services

With semantic conventions (OTel's standard):
ALL teams use: db.system, db.statement
→ ONE consistent naming scheme, dashboards and alerts
  work uniformly across every service, regardless of
  team or language
```

**Common semantic convention attributes:**
```
┌───────────────────────────────────────────────┐
│  http.method, http.status_code, http.route          │
│  db.system, db.statement, db.operation                 │
│  messaging.system, messaging.destination                  │
│  rpc.system, rpc.service, rpc.method                          │
└───────────────────────────────────────────────┘
```

---

## Metrics and Logs Signal Types

**Visual:**
```
Metrics Instrument Types:
┌───────────────────────────────────────────┐
│  Counter          → only goes UP (total requests,   │
│                   total errors)                          │
│  UpDownCounter       → can go up or down (active            │
│                   connections, queue depth)                     │
│  Histogram              → distribution of values (request           │
│                   duration, response size)                              │
│  Gauge                     → a point-in-time value (current                │
│                   memory usage, temperature)                                  │
└───────────────────────────────────────────────┘
```

```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)
request_counter = meter.create_counter("http.requests", description="Total HTTP requests")

request_counter.add(1, {"http.method": "GET", "http.status_code": 200})
```

**Visual — logs correlation:**
```
OpenTelemetry logs carry the SAME trace_id/span_id
as the active span when the log line was emitted:

{"timestamp": "10:15:32.110", "message": "Payment declined",
 "trace_id": "a1b2c3d4", "span_id": "00f067aa"}

This means you can jump DIRECTLY from a specific log
line to the exact trace/span that produced it — the
same log-to-trace correlation covered conceptually
in the Grafana/Loki notes, but with OTel providing
the standard mechanism that generates this link
automatically.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is debugging an intermittent issue where "checkout" traces sometimes appear as TWO separate, disconnected traces instead of one unified trace spanning the checkout service and the payment service.

**Root cause investigation:**
1. Confirms in Jaeger that SOME checkout requests correctly show one unified trace (checkout → payment-service, connected), while OTHERS show two completely separate traces with no parent-child relationship.
2. Investigates the payment-service call path for the BROKEN cases, and discovers this specific call path uses a raw `httpx` client configured with a **custom transport wrapper** that bypasses the auto-instrumentation's HTTP client patching — meaning the `traceparent` header was never being injected for THIS specific code path, while the standard `requests`-based calls elsewhere worked fine.
3. Fixes it by manually propagating context for this specific non-standard client, using OTel's `inject()` API directly:
```python
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # manually adds traceparent header
response = custom_httpx_client.post(url, headers=headers)
```
4. Confirms the fix by re-testing and seeing checkout → payment-service now consistently appears as ONE unified trace in Jaeger, regardless of which HTTP client code path is used.
5. Documents this as a known gotcha for the team: **auto-instrumentation only patches known, supported libraries** — any custom transport wrappers or less-common HTTP clients need manual context propagation added explicitly.

**Why this matters:** Broken context propagation is one of the most common OpenTelemetry issues in real codebases — it doesn't cause errors or crashes, just silently disconnected traces, making it easy to miss until someone notices "why does this request show up as two separate traces instead of one."

---

Next: **03instrumentation_practical.md** — practical auto vs manual instrumentation patterns, and real-world code examples across common scenarios.