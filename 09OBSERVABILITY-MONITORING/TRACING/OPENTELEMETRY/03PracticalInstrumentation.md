# OpenTelemetry - Practical Instrumentation

Hands-on patterns: when to rely on auto-instrumentation versus writing manual spans, and real-world code for common scenarios.

## Table of Contents
- [Auto-Instrumentation vs Manual Instrumentation](#auto-instrumentation-vs-manual-instrumentation)
- [Manual Span Creation Patterns](#manual-span-creation-patterns)
- [Instrumenting Database Calls](#instrumenting-database-calls)
- [Instrumenting Background Jobs/Queues](#instrumenting-background-jobsqueues)
- [Adding Business Context to Spans](#adding-business-context-to-spans)
- [Handling Errors and Exceptions](#handling-errors-and-exceptions)
- [Baggage: Passing Custom Context](#baggage-passing-custom-context)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Auto-Instrumentation vs Manual Instrumentation

**Visual:**
```
┌───────────────────────────────────────────────┐
│  Auto-Instrumentation                                  │
│  Covers: HTTP frameworks, database drivers,                 │
│  popular libraries (Flask, Express, psycopg2, etc.)             │
│  Effort: near-zero — just enable it                                │
│  Covers: "the request came in, called the DB,                        │
│  called an external API, sent a response"                                │
├───────────────────────────────────────────────┤
│  Manual Instrumentation                                     │
│  Covers: YOUR business logic — the specific                    │
│  steps and decisions unique to your application                    │
│  Effort: requires deliberately adding span code                        │
│  Covers: "validated the cart, calculated tax,                              │
│  applied a promo code, checked fraud rules"                                    │
└───────────────────────────────────────────────┘

Best practice: START with auto-instrumentation
(covers the "plumbing"), ADD manual spans only for
business logic steps you specifically want visibility
into — don't try to manually instrument everything.
```

---

## Manual Span Creation Patterns

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_order(order):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order.id)
        span.set_attribute("order.total", order.total)

        validate_order(order)
        charge_payment(order)
        send_confirmation(order)

def validate_order(order):
    with tracer.start_as_current_span("validate_order"):
        # validation logic
        pass
```

**Visual:**
```
Resulting Span Tree:
process_order
  ├── validate_order
  ├── charge_payment
  └── send_confirmation

Because each function opens its span using
"start_as_current_span" WHILE already inside the
parent's span context, OpenTelemetry automatically
figures out the parent-child relationship — you
never manually pass a "parent span ID" around.
```

**Using decorators for cleaner code:**
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def traced(span_name):
    def decorator(func):
        def wrapper(*args, **kwargs):
            with tracer.start_as_current_span(span_name):
                return func(*args, **kwargs)
        return wrapper
    return decorator

@traced("validate_order")
def validate_order(order):
    pass
```

---

## Instrumenting Database Calls

**Most database drivers have auto-instrumentation available — but understanding what it captures helps you know when to add more.**

```python
# Auto-instrumentation (via opentelemetry-instrumentation-psycopg2)
# automatically wraps this call with a span:
cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))
```

**Visual:**
```
What auto-instrumentation captures automatically:
┌─────────────────────────────────────┐
│  Span: "SELECT orders"                     │
│  db.system: postgresql                          │
│  db.statement: SELECT * FROM orders WHERE...       │
│  db.name: myapp_production                            │
│  net.peer.name: db.internal                                │
│  Duration: 12ms                                                │
└─────────────────────────────────────┘

What it DOESN'T capture (needs manual addition):
- WHY this query was run (business context)
- What happens if it returns zero rows vs many
- Custom retry logic around the query
```

**Adding manual context around an auto-instrumented call:**
```python
with tracer.start_as_current_span("fetch_user_orders") as span:
    cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))
    rows = cursor.fetchall()
    span.set_attribute("orders.count", len(rows))
    if len(rows) == 0:
        span.add_event("no_orders_found_for_user")
```

---

## Instrumenting Background Jobs/Queues

**Message queue processing is a common gap — the "request" doesn't arrive over HTTP, so context propagation needs explicit handling.**

**Visual:**
```
Producer side (publishing a message):
┌──────────┐  inject trace context  ┌──────────────┐
│  Web Request  │ ──────────────────────────→ │  Message Queue     │
│  (has active     │    into message headers/     │  (Kafka/RabbitMQ)    │
│   trace context)   │    metadata                     │                        │
└──────────┘                                └──────────────┘

Consumer side (processing the message):
┌──────────────┐  extract trace context  ┌──────────┐
│  Message Queue     │ ────────────────────────────→ │  Background   │
│                        │    from message metadata       │  Worker         │
└──────────────┘                                    └──────────┘

Result: the background job's processing span connects
BACK to the original web request's trace — even though
significant time may have passed and it's running in
a completely different process.
```

**Producer code:**
```python
from opentelemetry.propagate import inject

def publish_order_event(order):
    headers = {}
    inject(headers)  # injects traceparent into headers dict
    queue.publish(
        message={"order_id": order.id},
        headers=headers
    )
```

**Consumer code:**
```python
from opentelemetry.propagate import extract
from opentelemetry import context

def process_message(message):
    ctx = extract(message.headers)
    token = context.attach(ctx)
    try:
        with tracer.start_as_current_span("process_order_event"):
            handle_order(message.body)
    finally:
        context.detach(token)
```

**Visual:**
```
Why this matters:
Without explicit propagation, a trace for "user places
order" would end the moment the HTTP response was sent —
the ASYNCHRONOUS background processing (sending emails,
updating inventory, etc.) would appear as a completely
separate, disconnected trace, making it hard to see
the FULL picture of everything one user action triggered.
```

---

## Adding Business Context to Spans

**The most valuable manual instrumentation captures business-meaningful data, not just technical timing.**

```python
with tracer.start_as_current_span("apply_discount") as span:
    discount = calculate_discount(order, promo_code)
    span.set_attribute("promo.code", promo_code)
    span.set_attribute("promo.discount_percent", discount.percent)
    span.set_attribute("promo.valid", discount.is_valid)

    if not discount.is_valid:
        span.add_event("promo_code_rejected", {"reason": discount.rejection_reason})
```

**Visual:**
```
Why this transforms debugging:
Generic technical span: "apply_discount took 45ms" —
   tells you TIMING but nothing about WHAT happened

Business-context-rich span: "apply_discount: promo
   code SUMMER20 REJECTED because expired" — immediately
   tells you WHY a customer's discount didn't apply,
   without needing to reproduce the issue or dig through
   logs separately.

This is often MORE valuable than the timing data itself
for day-to-day debugging of business logic issues.
```

---

## Handling Errors and Exceptions

```python
from opentelemetry.trace import Status, StatusCode

with tracer.start_as_current_span("charge_payment") as span:
    try:
        result = payment_gateway.charge(amount)
    except PaymentDeclinedError as e:
        span.set_status(Status(StatusCode.ERROR, str(e)))
        span.record_exception(e)
        raise
```

**Visual:**
```
span.record_exception(e) captures:
┌─────────────────────────────────────┐
│  exception.type: PaymentDeclinedError      │
│  exception.message: "Insufficient funds"      │
│  exception.stacktrace: (full traceback)          │
└─────────────────────────────────────┘

This attaches the FULL exception detail directly to
the span as an event — meaning when you find a failed
trace in Jaeger, the exact exception and stack trace
that caused it is right there, without needing to
separately search application logs for the same timestamp.
```

---

## Baggage: Passing Custom Context

**Baggage propagates arbitrary key-value data ACROSS service boundaries, alongside the trace context itself.**

```python
from opentelemetry import baggage, context

ctx = baggage.set_baggage("user.tier", "premium")
context.attach(ctx)

# Later, in a DIFFERENT service, after propagation:
user_tier = baggage.get_baggage("user.tier")  # "premium"
```

**Visual:**
```
Why Baggage is different from a span attribute:
Span attribute → describes ONE specific span, doesn't
                automatically propagate to child services

Baggage           → propagates ACROSS every downstream
                service call, available to ALL of them

Use case: tag a request as coming from a "premium"
tier user at the very start, and have EVERY downstream
service (recommendation engine, pricing service, etc.)
automatically have access to that context WITHOUT
needing to look it up again themselves.
```

⚠️ **Caution:** Baggage is propagated in HTTP headers on every single call — putting large or numerous values in Baggage adds overhead to every request in the chain. Use sparingly, for small, genuinely cross-cutting context.

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is helping a team debug why certain orders take much longer to process than others, with no clear pattern visible in existing dashboards.

**What they do:**
1. Confirms **auto-instrumentation** is already capturing HTTP and database call timing, but notices the actual business logic (`calculate_discount`, `apply_fraud_check`, `select_shipping_method`) is completely invisible — appearing as one large, undifferentiated block of time inside the parent span.
2. Adds **manual spans** around each of these three business logic steps, including business-relevant attributes (`promo.code`, `fraud_check.risk_score`, `shipping.method`).
3. Within a day of deploying this change, traces clearly show that orders using a SPECIFIC promo code consistently spend an extra 800ms in `calculate_discount` — tracing this further reveals that promo code's discount rule makes an unnecessary external API call that other promo codes don't require.
4. Separately, notices background job processing (order confirmation emails) appears as completely disconnected traces from the original checkout request, and fixes this by adding explicit **context injection/extraction** around the message queue producer/consumer, unifying the full order lifecycle into one traceable flow.
5. Adds **exception recording** to the fraud-check span specifically, since that step had a history of silent failures that were previously only visible by grepping application logs after the fact.

**Why this matters:** Auto-instrumentation alone gives you the "plumbing" view (HTTP/DB timing) but the actual business logic — often where real, hard-to-explain performance issues hide — only becomes visible once someone deliberately adds manual spans with meaningful business context around it.

---

Next: **04collector_pipelines.md** — deep dive into the OpenTelemetry Collector's receivers, processors, and exporters, and real-world deployment patterns.