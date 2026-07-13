# Loki - Introduction & Architecture

Understanding what Loki is, why it exists, and how its "index only the labels, not the text" design makes it fundamentally different from other log systems.

## Table of Contents
- [What is Loki](#what-is-loki)
- [The "Like Prometheus, but for Logs" Philosophy](#the-like-prometheus-but-for-logs-philosophy)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [Loki vs Alternatives](#loki-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is Loki

**Loki is a log aggregation system built by Grafana Labs, designed to be cost-efficient and easy to operate by indexing only metadata (labels), not the full text of every log line.**

**Visual:**
```
Application Logs (scattered across servers/pods)
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Pod 1        │ │  Pod 2        │ │  Pod 3        │
│  app.log        │ │  app.log        │ │  app.log        │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │             │             │
     └─────────────┴─────────────┘
                   ↓  shipped by an agent (Promtail/Alloy)
            ┌──────────────┐
            │      Loki         │
            │  (centralized,      │
            │   labeled, queryable)│
            └──────────────┘
                   ↓
            ┌──────────────┐
            │    Grafana        │
            │  (query & view      │
            │   via LogQL)          │
            └──────────────┘
```

---

## The "Like Prometheus, but for Logs" Philosophy

**This is the single most important concept for understanding Loki's design.**

**Visual:**
```
Traditional Log Systems (e.g. Elasticsearch/ELK):
Every single WORD in every log line gets indexed (full-text index)
┌────────────────────────────────────────┐
│  Log: "User 4821 login failed from IP..."  │
│  Indexed: User, 4821, login, failed, from,   │
│           IP, ... (every token indexed)         │
└────────────────────────────────────────┘
→ Powerful full-text search, but the INDEX itself
  becomes enormous — often larger than the raw logs.

Loki's Approach:
Only LABELS (metadata) get indexed — the log line
content itself is stored compressed, unindexed.
┌────────────────────────────────────────┐
│  Labels indexed: {app="payments",            │
│                    env="production",             │
│                    pod="payments-7f8d9"}          │
│  Log line content: stored as compressed TEXT,       │
│                    searched via grep-like scan         │
│                    ONLY within matching label streams     │
└────────────────────────────────────────┘
→ Tiny index, dramatically cheaper storage,
  at the cost of full-text search being a linear
  scan (mitigated by narrowing labels first).
```

**Visual — why this tradeoff makes sense in practice:**
```
Real query pattern in an incident:
"Show me ERROR logs from the payments service, production,
 in the last 15 minutes"
       ↓
{app="payments", env="production"} |= "ERROR"
       ↓
Step 1: Labels narrow it to a SMALL set of log streams
         (fast, index-based lookup)
Step 2: Text search "ERROR" only within that small,
         already-narrowed set (fast, because it's not
         scanning the entire org's logs)

You rarely need to full-text search EVERY log ever
written across the whole company — you almost always
know at least the service/environment first.
```

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                                How Loki Helps
──────────────────────────────────────────────────────────────────
Log storage costs spiraling (ELK)        Much smaller index, cheaper
                                        object storage (S3-compatible) for chunks
Logs and metrics live in separate tools    Native integration with Grafana,
                                        same query bar style as PromQL
Complex to operate (ELK cluster tuning)     Simpler operational model,
                                        horizontally scalable microservices
Need to correlate logs with metrics          Same time range, same dashboard,
during incidents                          same tool as Prometheus/Grafana
```

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                       Loki Concepts                       │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Log Stream        →  A unique combination of label          │
│                       values (e.g. app=payments, pod=x1)         │
│                                                           │
│  Label              →  Key-value metadata attached to a           │
│                       stream (indexed, used for filtering)          │
│                                                           │
│  Chunk              →  Compressed blob of actual log line             │
│                       content for a stream, stored in object object   │
│                       storage (S3/GCS/etc.)                              │
│                                                           │
│  LogQL              →  Loki's query language, similar in style           │
│                       to PromQL but for logs                                │
│                                                           │
│  Agent (Promtail/    →  The component that reads logs from                    │
│   Alloy/Fluent Bit)     files/containers and ships them to Loki                  │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
┌──────────┐   reads logs   ┌──────────────────┐
│  Node/Pod    │ ─────────────→ │  Promtail (Agent)     │
│  (log files)   │                 │  (adds labels, tails    │
└──────────┘                 │   log files)              │
                              └─────────┬──────────────┘
                                        │ pushes logs
                                        ↓
                          ┌───────────────────────────┐
                          │         Loki Server              │
                          │  ┌───────────────────────┐  │
                          │  │  Distributor                 │  │  ← receives, validates,
                          │  ├───────────────────────┤  │     hashes to ingesters
                          │  │  Ingester                       │  │  ← buffers, compresses
                          │  ├───────────────────────┤  │     into chunks
                          │  │  Querier                        │  │  ← handles LogQL queries
                          │  ├───────────────────────┤  │
                          │  │  Index (labels only)              │  │  ← small, fast lookup
                          │  └───────────────────────┘  │
                          └─────────────┬─────────────────┘
                                        │  chunks stored in
                                        ↓
                          ┌───────────────────────────┐
                          │   Object Storage (S3/GCS/etc)   │  ← the bulk of the data,
                          └───────────────────────────┘     cheap, durable
```

**Flow in words:**
1. **Promtail** (or another agent) tails log files/container output, attaches **labels** (e.g., pod name, namespace, app), and pushes to Loki.
2. The **Distributor** receives incoming logs and routes them to the right **Ingester** based on label hash.
3. The **Ingester** buffers recent logs in memory, periodically flushing compressed **chunks** to object storage.
4. The **Index** (small, label-only) tracks which chunks belong to which streams.
5. The **Querier** handles LogQL queries — using the index to find relevant chunks, then scanning chunk content for text filters.

---

## Loki vs Alternatives

**Visual:**
```
Tool                Focus                          Notes
─────────────────────────────────────────────────────────────────────
Loki                  Label-indexed, cost-efficient    Best when paired with Grafana +
                                                        Prometheus, simpler ops model
Elasticsearch/ELK        Full-text indexed everything      Powerful search, but expensive
                                                        storage/index, complex to operate
Splunk                    Enterprise full-text + analytics  Commercial, feature-rich, costly
CloudWatch Logs             AWS-native                          Simple, but AWS-locked-in,
                                                        pricier at high volume
Fluentd/Fluent Bit             Log SHIPPERS, not storage           Often used to SEND logs
                                                        into Loki/ES/Splunk, not a
                                                        replacement for any of them

Loki's niche: cheapest storage cost per GB of logs,
tightly integrated with Grafana/Prometheus, ideal when
you already know your key filtering dimensions (service,
namespace, environment) rather than needing arbitrary
full-text search across everything.
```

---

## Real-Life DevOps Use Case

**Scenario:** A company running Elasticsearch for logs is facing a ballooning cloud bill — the log cluster alone costs more than all their compute combined, due to the full-text index size.

**What the DevOps engineer does:**
1. Evaluates that **90% of log queries** in practice start with "show me logs from service X, in environment Y" — meaning the team is barely using ELK's arbitrary full-text search across the *entire* dataset; they almost always already know the service/environment first.
2. Migrates to **Loki**, deploying it alongside their existing **Prometheus + Grafana** stack, using **Promtail** (later Grafana Alloy) as the log-shipping agent on every Kubernetes node.
3. Defines labels carefully: `namespace`, `app`, `pod` — deliberately keeping cardinality low (avoiding labels like `user_id` or `request_id`, covered in depth in file 02).
4. Points Loki's chunk storage at **S3**, which costs a fraction of the equivalent Elasticsearch cluster's disk/compute footprint.
5. Result: **log storage costs drop significantly**, log queries during incidents are just as fast (since engineers already knew the service/environment to filter by), and logs now appear in the **same Grafana dashboards** as their metrics — enabling direct correlation without switching tools.

**Result:** Loki isn't a strict full-text-search replacement for every ELK use case — but for the common "filter by known service/environment, then search text" pattern that dominates real DevOps incident response, it delivers similar practical value at dramatically lower cost and operational complexity.

---

Next: **01installation_and_setup.md** — installing Loki, setting up Promtail, and shipping your first logs.