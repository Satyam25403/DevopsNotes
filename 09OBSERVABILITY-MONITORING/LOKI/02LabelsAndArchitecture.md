# Loki - Labels & Architecture Deep Dive

The single most important theory for operating Loki well: understanding labels, cardinality, and what each internal component actually does.

## Table of Contents
- [Labels Are Everything](#labels-are-everything)
- [The Cardinality Problem](#the-cardinality-problem)
- [Good Labels vs Bad Labels](#good-labels-vs-bad-labels)
- [Streams and Chunks](#streams-and-chunks)
- [Component Deep Dive](#component-deep-dive)
- [The Index vs Chunk Storage Split](#the-index-vs-chunk-storage-split)
- [Structured Metadata (Loki 3.x)](#structured-metadata-loki-3x)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Labels Are Everything

**Every design decision in Loki flows from one rule: labels are indexed, log content is not.**

**Visual:**
```
Log Stream Identity:
{namespace="production", app="payments", pod="payments-7f8d9"}
        ↑                     ↑                  ↑
    label 1               label 2            label 3

This EXACT combination of label values = ONE stream.
A different pod name (payments-9a1b2) = a DIFFERENT stream,
even though it's "the same app."

All log lines sharing the same label combination are
stored together, compressed, as that stream's chunks.
```

---

## The Cardinality Problem

**This is the #1 way Loki deployments get into trouble — and the most important thing to understand before using it in production.**

**Visual:**
```
Low Cardinality Label (GOOD):
namespace: production, staging, development
→ only 3 possible values → 3 streams max, contributed by this label

High Cardinality Label (DANGEROUS):
user_id: 4821, 4822, 4823, ... (millions of unique users)
→ millions of possible values → if used as a LABEL,
  creates MILLIONS of tiny streams
```

**Visual — why high cardinality breaks Loki:**
```
Intended Design:
Few streams, each with LOTS of log lines
┌─────────────────────────┐
│ Stream: {app=payments}      │
│  line1, line2, line3, ... 50,000 lines│
└─────────────────────────┘
→ Small index (one entry per stream),
  efficient chunk compression

What Happens With user_id as a Label:
MANY streams, each with almost NO log lines
┌───────┐┌───────┐┌───────┐┌───────┐
│{user=1}││{user=2}││{user=3}││{user=4}│ ...millions more
│1 line   ││1 line   ││1 line   ││1 line   │
└───────┘└───────┘└───────┘└───────┘
→ HUGE index (millions of entries),
  tiny inefficient chunks, memory pressure,
  slow queries, potential ingester crashes

This is called "cardinality explosion" and is the
single most common cause of Loki outages/performance
problems in the real world.
```

---

## Good Labels vs Bad Labels

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Good Labels (low cardinality, use as LABELS)            │
├───────────────────────────────────────────────────┤
│  namespace         (10s of values)                        │
│  app / service        (10s-100s of values)                    │
│  environment            (a handful: prod/staging/dev)            │
│  pod (with caution)        (bounded by replica count,               │
│                          acceptable in most clusters)                  │
│  log_level                (debug/info/warn/error — tiny set)             │
└───────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────┐
│ Bad Labels (high cardinality, DO NOT use as labels)      │
├───────────────────────────────────────────────────┤
│  user_id                    (potentially millions)             │
│  request_id / trace_id          (unique per request, essentially   │
│                              infinite cardinality)                     │
│  IP address                    (many thousands+)                        │
│  email address                    (essentially infinite)                   │
│  full URL with query params          (near-infinite combinations)            │
└───────────────────────────────────────────────────┘
```

**Visual — the correct place for high-cardinality data:**
```
WRONG: attach user_id as a LABEL
{app="payments", user_id="4821"} log line text...
→ cardinality explosion

RIGHT: keep user_id IN the log line content itself,
       search for it with a text/JSON filter instead
{app="payments"} | json | user_id="4821"
→ ONE stream ({app="payments"}), filtered by CONTENT,
  not by label — no cardinality impact at all
```

---

## Streams and Chunks

**Visual:**
```
┌─────────────────────────────────────────────┐
│  Stream: {app="payments", pod="payments-7f8d9"}  │
│                                                  │
│  Chunk 1 (flushed, compressed, in object storage) │
│  Chunk 2 (flushed, compressed, in object storage) │
│  Chunk 3 (currently being written, in memory)       │
└─────────────────────────────────────────────┘

A Chunk is:
- A time-ordered, compressed block of log lines
  belonging to ONE stream
- Flushed to object storage after reaching a size/time
  limit (e.g. 1MB or 30 minutes, whichever first)
- Immutable once flushed
```

**Visual — why chunking matters for compression:**
```
Log lines from the SAME stream tend to be very similar
in structure (same app, same log format) →
compress extremely well together (gzip/snappy)

This is why Loki achieves such good compression ratios —
grouping genuinely similar text together, rather than
mixing unrelated logs from different apps/services
into the same chunk.
```

---

## Component Deep Dive

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Component        Responsibility                            │
├─────────────────────────────────────────────────────┤
│ Distributor          Receives incoming log pushes, validates,  │
│                     hashes stream labels to determine which     │
│                     ingester should own that stream               │
│                                                          │
│ Ingester              Buffers recent log data in memory for a      │
│                     stream, periodically flushes completed           │
│                     chunks to object storage                            │
│                                                          │
│ Querier                Executes LogQL queries — checks the index       │
│                     for matching streams, fetches relevant                │
│                     chunks (from ingester memory OR object                   │
│                     storage), applies text/filter operations                   │
│                                                          │
│ Query Frontend           Splits large queries into smaller sub-               │
│                     queries, queues and parallelizes them,                       │
│                     improves performance for big time ranges                       │
│                                                          │
│ Compactor                Runs in the background, merges/                            │
│                     deduplicates index files, applies                                  │
│                     retention policies (deletes old data)                                 │
└─────────────────────────────────────────────────────┘
```

**Visual — the write path:**
```
Promtail push → Distributor (validates, hashes) →
   Ingester (buffers in memory) → [time/size limit reached] →
   Chunk flushed to Object Storage (S3/GCS/etc.)
                                        ↓
                              Index updated with chunk location
```

**Visual — the read path:**
```
LogQL query → Query Frontend (splits into sub-queries) →
   Querier(s) → checks Index for matching streams →
   fetches chunks from Ingester memory (recent data) AND/OR
   Object Storage (older data) → applies |= filters →
   merges and returns results
```

---

## The Index vs Chunk Storage Split

**Visual:**
```
┌─────────────────────────────┐  ┌─────────────────────────────┐
│         INDEX                    │  │      CHUNK STORAGE                │
│  (small, fast, frequently          │  │  (large, cheap, object storage:     │
│   queried)                            │  │   S3/GCS/Azure Blob)                    │
│                                        │  │                                          │
│  Maps: labels → which chunks             │  │  Actual compressed log line               │
│  exist for that stream, and                │  │  content, referenced by the                │
│  their time ranges                            │  │  index but stored separately                  │
└─────────────────────────────┘  └─────────────────────────────┘

Why split them?
Index needs to be FAST (queried on every single request)
Chunks need to be CHEAP (bulk of the data volume, accessed
   less frequently, don't need expensive fast storage)

This split is exactly what keeps Loki's cost profile so
much lower than full-text-indexed systems — you pay premium
storage costs ONLY for the small index, not for the bulk
log content itself.
```

---

## Structured Metadata (Loki 3.x)

**A newer feature addressing a real pain point: sometimes you DO want to filter on something like `trace_id`, without paying the cardinality cost of a label.**

**Visual:**
```
Structured Metadata:
Attached to individual log lines (not stream labels),
indexed separately in a way that doesn't cause the
stream-explosion problem regular labels would.

{app="payments"} log line, with structured metadata:
   trace_id=abc123, request_id=xyz789

Query:
{app="payments"} | trace_id="abc123"
→ Efficiently filters by trace_id WITHOUT creating
  a separate stream per trace_id (unlike using it as
  a true label would).
```

This gives teams a middle ground: high-cardinality fields can still be efficiently queried, just not via the stream-label mechanism.

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer inherits a Loki deployment that's frequently running out of memory on ingesters and has become unreliable.

**Root cause investigation:**
1. Checks the **Promtail scrape configs** across the fleet and discovers a well-meaning developer added `request_id` as a label to help with debugging a specific incident months ago — and it was never removed.
2. Confirms via Loki's own metrics (`loki_ingester_memory_streams`) that the number of active streams is in the millions — far beyond what a handful of services with reasonable labels (`app`, `namespace`, `pod`) should ever produce.
3. Identifies this as a classic **cardinality explosion**: `request_id` is unique per request, so effectively every single log line was creating a brand-new stream instead of joining an existing one.
4. **Fixes it** by removing `request_id` from the label set entirely, migrating that debugging need to **Structured Metadata** instead (Loki 3.x) — preserving the ability to filter by `request_id` without the per-request stream explosion.
5. After redeploying the corrected Promtail config, ingester memory usage drops dramatically within hours as old high-cardinality streams age out, and query performance improves noticeably.

**Why this matters:** Cardinality problems are the single most common reason Loki deployments become unreliable in production — and they're entirely preventable by understanding, before deploying anything, which fields are safe to use as labels and which belong in log content or structured metadata instead.

---

Next: **03logql_practical.md** — hands-on LogQL: filters, parsers, metric queries, and patterns used constantly in real dashboards and alerts.