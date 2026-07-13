# Loki - LogQL Practical Guide

Hands-on LogQL: the query language that turns raw log streams into filtered results, parsed fields, and even Prometheus-style metrics.

## Table of Contents
- [LogQL Query Anatomy](#logql-query-anatomy)
- [Label Selectors](#label-selectors)
- [Line Filter Expressions](#line-filter-expressions)
- [Parser Expressions](#parser-expressions)
- [Label Filter Expressions](#label-filter-expressions)
- [Line Format and Label Format](#line-format-and-label-format)
- [Metric Queries](#metric-queries)
- [Unwrap Expressions](#unwrap-expressions)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## LogQL Query Anatomy

**Visual:**
```
{app="payments"} |= "ERROR" | json | status_code="500" | line_format "{{.message}}"
└──────┬───────┘ └────┬───┘ └──┬─┘ └──────┬───────┘ └───────────┬───────────┘
   Label Selector    Line       Parser    Label Filter          Formatting
   (which streams)    Filter    (extract   (filter on            (change display)
                     (text match) fields)   parsed fields)

Every LogQL query starts with a Label Selector,
then optionally CHAINS additional operations —
each one processing the output of the previous stage.
```

---

## Label Selectors

**The mandatory first part of every LogQL query — always in curly braces.**

```logql
{app="payments"}
{app="payments", env="production"}
{app=~"payments|auth"}
{app!="payments"}
```

**Visual:**
```
┌──────┬───────────────────────────────────┐
│  =     │  Exact match                          │
│  !=    │  Not equal                             │
│  =~    │  Regex match (matches ANY of a list)      │
│  !~    │  Regex does NOT match                       │
└──────┴───────────────────────────────────┘

{app=~"payments|auth"}
→ selects streams where app is EITHER "payments" OR "auth"
```

---

## Line Filter Expressions

**Text-based filtering applied AFTER label selection — this is the "grep" part of LogQL.**

```logql
{app="payments"} |= "ERROR"
{app="payments"} != "DEBUG"
{app="payments"} |~ "error|exception|fatal"
{app="payments"} !~ "healthcheck"
```

**Visual:**
```
┌──────┬───────────────────────────────────┐
│  |=    │  Line CONTAINS this string              │
│  !=    │  Line does NOT contain this string         │
│  |~    │  Line matches this REGEX                     │
│  !~    │  Line does NOT match this regex                │
└──────┴───────────────────────────────────┘

Chaining multiple filters:
{app="payments"} |= "ERROR" != "healthcheck"
→ Contains "ERROR" AND does NOT contain "healthcheck"
  (filters out noisy healthcheck error logs)
```

**Visual — filter order matters for performance:**
```
Best Practice: put the MOST SELECTIVE filter first
{app="payments"} |= "ERROR" |= "timeout"
→ First narrows to lines with "ERROR" (likely fewer),
  THEN further narrows to those also containing "timeout" —
  cheaper than scanning for "timeout" across all lines first.
```

---

## Parser Expressions

**Extracts structured fields OUT of unstructured or semi-structured log lines, so you can filter/format on them.**

### JSON Parser

```logql
{app="payments"} | json
```

**Visual:**
```
Raw log line:
{"level":"error","status_code":500,"message":"payment failed"}

After | json:
Extracted fields (now usable like labels, but NOT indexed):
  level = error
  status_code = 500
  message = payment failed

{app="payments"} | json | status_code="500"
→ Now filters on the PARSED field, not raw text matching
```

### Logfmt Parser

```logql
{app="payments"} | logfmt
```

**Visual:**
```
Raw log line (logfmt style):
level=error status_code=500 msg="payment failed"

After | logfmt: same extraction concept as JSON,
but for the logfmt key=value format instead.
```

### Regexp/Pattern Parser

```logql
{app="payments"} | pattern "<ip> - - [<timestamp>] \"<method> <path>\""
```

**Visual:**
```
Raw log line (Apache-style access log):
192.168.1.5 - - [12/Jul/2026:10:15:32] "GET /api/orders"

After | pattern:
  ip = 192.168.1.5
  timestamp = 12/Jul/2026:10:15:32
  method = GET
  path = /api/orders

Useful for logs that AREN'T JSON/logfmt but follow a
consistent, parseable text structure.
```

---

## Label Filter Expressions

**Filter on fields AFTER a parser has extracted them (numeric/string comparisons).**

```logql
{app="payments"} | json | status_code >= 500
{app="payments"} | json | duration > 2s
{app="payments"} | json | status_code="500", env="production"
```

**Visual:**
```
Why this is different from a line filter:
Line filter (|=)   → text-based, string "contains" only
Label filter (after| json) → TYPED comparisons: numeric (>=, >, <),
                     duration-aware (2s, 500ms), and exact match

{app="payments"} | json | duration > 2s
→ Only lines where the parsed "duration" field, interpreted
  as a time duration, exceeds 2 seconds — impossible with
  plain text filtering alone.
```

---

## Line Format and Label Format

**Reshape what's displayed, useful for making noisy JSON readable in a dashboard panel.**

```logql
{app="payments"} | json | line_format "{{.timestamp}} [{{.level}}] {{.message}}"
```

**Visual:**
```
Before (raw JSON, hard to scan visually):
{"timestamp":"10:15:32","level":"error","message":"payment failed","trace_id":"abc123","user_agent":"..."}

After line_format:
10:15:32 [error] payment failed

Much easier for a human to scan a Logs panel full of
these lines during an incident, without the JSON noise.
```

---

## Metric Queries

**LogQL's most powerful feature: turning log lines into NUMBERS over time, usable exactly like a Prometheus metric.**

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Function                Purpose                        │
├───────────────────────────────────────────────────┤
│ rate(...[5m])              Per-second rate of log lines     │
│                          matching a selector/filter            │
│                                                        │
│ count_over_time(...[5m])     Raw count of matching lines           │
│                          in the time window                          │
│                                                        │
│ bytes_rate(...[5m])            Rate of bytes ingested (useful          │
│                          for spotting abnormal log volume)                │
│                                                        │
│ sum by (label) (...)              Aggregate across streams,                 │
│                          grouped by a label                                    │
└───────────────────────────────────────────────────┘
```

**Example: error rate as a metric, usable in a Time Series panel:**
```logql
sum(rate({app="payments"} |= "ERROR" [5m]))
```

**Visual:**
```
{app="payments"} |= "ERROR"    ← raw matching log lines
        ↓
rate(...[5m])                    ← convert to per-second rate,
                                    over rolling 5-minute windows
        ↓
sum(...)                          ← total across all pods/instances
                                    of the payments service
        ↓
Result: a single NUMBER over time — plot it in a
Time Series panel exactly like a Prometheus metric,
and it can even be used in a Grafana ALERT RULE
(covered in the Grafana alerting notes).
```

**Grouping by label:**
```logql
sum by (pod) (rate({app="payments"} |= "ERROR" [5m]))
```

**Visual:**
```
Returns a SEPARATE line per pod:
  payments-7f8d9: 0.5 errors/sec
  payments-9a1b2: 12.3 errors/sec   ← this pod is clearly unhealthy
  payments-3c4d5: 0.6 errors/sec

Immediately pinpoints WHICH specific instance is
misbehaving, rather than just an aggregate total.
```

---

## Unwrap Expressions

**Extracts a NUMERIC VALUE from within the log line itself (not just counting matching lines) for more precise metric queries.**

```logql
quantile_over_time(0.99, {app="payments"} | json | unwrap duration [5m])
```

**Visual:**
```
Raw log lines contain an actual duration value:
{"duration": 0.234}
{"duration": 1.502}
{"duration": 0.089}

| json                → parses the JSON
| unwrap duration        → extracts "duration" as a NUMBER
                           (not just counting lines)
quantile_over_time(0.99, ...[5m])
                       → calculates the P99 duration from
                         the actual VALUES in the logs

This effectively derives a latency percentile metric
straight from log data, without needing a separate
Prometheus histogram metric instrumented in the app.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer needs to build a Grafana panel showing the payments service's error rate trending over time, plus be able to drill into exactly which pod is causing spikes — using ONLY existing JSON logs, without adding new Prometheus instrumentation to the app.

**What they build:**
1. A **metric query panel**: `sum(rate({app="payments"} | json | level="error" [5m]))` — turning the JSON `level` field into a time-series error rate, avoiding the earlier mistake of using a fragile plain-text `|= "error"` filter that might also match unrelated words like "error_count: 0".
2. A **secondary breakdown panel**: `sum by (pod) (rate({app="payments"} | json | level="error" [5m]))` — showing per-pod error rates side-by-side, so an on-call engineer can immediately spot if the errors are isolated to one misbehaving pod versus a systemic issue.
3. A **Logs panel** just below, using `{app="payments"} | json | level="error" | line_format "{{.timestamp}} {{.message}}"` — showing the human-readable message only, stripped of JSON noise, for quick visual scanning during an incident.
4. A **latency panel** using `quantile_over_time(0.99, {app="payments"} | json | unwrap duration [5m])` — deriving a P99 latency trend directly from the existing log field, without needing the application team to add a new Prometheus histogram metric first (a multi-sprint ask they didn't have time for).

**Why this matters:** LogQL's parser and unwrap capabilities let a DevOps engineer derive genuinely useful *metrics* (error rates, latency percentiles) directly from logs that already exist — often faster to ship than waiting on application code changes to add proper Prometheus instrumentation, even though native metrics remain the better long-term solution.

---

Next: **04log_ingestion_pipelines.md** — practical Promtail/Alloy configuration for Kubernetes, relabeling, multi-source log ingestion, and parsing pipelines.