# OpenTelemetry - Introduction & Architecture

Understanding what OpenTelemetry is, the vendor lock-in problem it solves, and how it unifies traces, metrics, and logs under one standard.

## Table of Contents
- [What is OpenTelemetry](#what-is-opentelemetry)
- [The Vendor Lock-In Problem](#the-vendor-lock-in-problem)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [The Three Pillars of Observability](#the-three-pillars-of-observability)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [OpenTelemetry vs Alternatives](#opentelemetry-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is OpenTelemetry

**OpenTelemetry (OTel) is a vendor-neutral, open-source standard for generating, collecting, and exporting telemetry data — traces, metrics, and logs — from your applications.**

**Visual:**
```
Your Application Code
┌──────────────────────┐
│  OpenTelemetry SDK          │  ← instruments your code ONCE,
│  (traces, metrics, logs)       │     using an open standard API
└──────────┬───────────────┘
           │  exports telemetry in a
           │  STANDARD format (OTLP)
           ↓
┌──────────────────────┐
│   OpenTelemetry Collector      │  ← receives, processes, routes
└──────────┬───────────────┘
           │
    ┌──────┼──────┬──────────┐
    ↓             ↓             ↓
┌────────┐  ┌────────┐  ┌────────┐
│ Jaeger        │  │ Prometheus    │  │ Datadog        │  ← send to ANY backend,
│ (traces)        │  │ (metrics)       │  │ (all-in-one)      │     swap freely later
└────────┘  └────────┘  └────────┘
```

---

## The Vendor Lock-In Problem

**This is the exact problem OpenTelemetry was created to solve.**

**Visual:**
```
Before OpenTelemetry (proprietary instrumentation):
┌─────────────────────────────────────┐
│  App instrumented using Datadog's SDK      │
│  (Datadog-specific annotations, agents)         │
└─────────────────────────────────────┘
        ↓
Company decides to switch to New Relic (cost, features)
        ↓
EVERY service must be RE-INSTRUMENTED with New Relic's
SDK instead — a massive, risky, all-or-nothing migration
project touching every single codebase

With OpenTelemetry (vendor-neutral instrumentation):
┌─────────────────────────────────────┐
│  App instrumented using OpenTelemetry's      │
│  standard SDK (vendor-agnostic)                  │
└─────────────────────────────────────┘
        ↓
Company decides to switch backends
        ↓
Change ONE line of Collector configuration
(which exporter to send data to) —
APPLICATION CODE NEVER CHANGES AT ALL
```

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                                How OpenTelemetry Helps
──────────────────────────────────────────────────────────────────
Locked into one observability vendor      Instrument once, export anywhere —
                                        switch backends without code changes
Different teams use different tracing       ONE standard API/SDK across every
libraries inconsistently                  language, consistent instrumentation
Traces, metrics, and logs live in            Unified data model correlates all
separate, disconnected tools                three signal types via shared context
Expensive commercial APM tools                Pairs with free/open-source backends
                                        (Jaeger, Prometheus, Grafana Tempo)
No control over telemetry pipeline               The Collector lets you filter, sample,
before it reaches a vendor                    and transform data before it's sent anywhere
```

---

## The Three Pillars of Observability

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│                                                           │
│  TRACES        →  The journey of a SINGLE request as it      │
│               flows through multiple services                    │
│               "This request took 340ms, and here's                 │
│                EXACTLY which service call was slow"                    │
│                                                           │
│  METRICS         →  Aggregated NUMBERS over time                        │
│               "Error rate is 2%, P99 latency is 450ms"                     │
│                                                           │
│  LOGS               →  Discrete, timestamped EVENTS with                      │
│               context (often correlated to a trace)                              │
│               "At 10:15:32, payment failed: insufficient funds"                     │
│                                                           │
└─────────────────────────────────────────────────────────┘

OpenTelemetry's key innovation: unifying instrumentation
for ALL THREE under one SDK and one context-propagation
mechanism, so a trace ID can link a specific log line to
the specific request span that produced it.
```

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                 OpenTelemetry Concepts                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Span               →  A single unit of work within a trace    │
│                       (e.g. "call the database")                   │
│                                                           │
│  Trace                 →  A collection of spans representing         │
│                       one complete request's journey                    │
│                                                           │
│  Context Propagation      →  Passing trace identity ACROSS service         │
│                       boundaries (HTTP headers, etc.)                         │
│                                                           │
│  Resource                    →  Metadata describing WHERE telemetry               │
│                       came from (service name, host, version)                        │
│                                                           │
│  Instrumentation                →  Code (auto or manual) that generates                  │
│                       spans/metrics/logs                                                    │
│                                                           │
│  Collector                          →  A separate process that receives,                       │
│                       processes, and exports telemetry                                            │
│                                                           │
│  OTLP                                  →  OpenTelemetry Protocol — the                            │
│                       standard wire format for sending data                                          │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
┌───────────────────────────┐
│       Application Code            │
│  ┌───────────────────────┐  │
│  │  OTel SDK (API + SDK)       │  │  ← generates spans/metrics/logs
│  └───────────┬───────────┘  │
└──────────────┼───────────────┘
               │  OTLP (gRPC/HTTP)
               ↓
┌───────────────────────────┐
│      OpenTelemetry Collector       │
│  ┌───────────────────────┐  │
│  │  Receivers                    │  │  ← accepts data (OTLP, Jaeger, etc.)
│  ├───────────────────────┤  │
│  │  Processors                     │  │  ← batch, filter, sample, enrich
│  ├───────────────────────┤  │
│  │  Exporters                        │  │  ← sends to one or more backends
│  └───────────┬───────────┘  │
└──────────────┼───────────────┘
               │
      ┌────────┼────────┬──────────┐
      ↓                    ↓             ↓
┌──────────┐        ┌──────────┐  ┌──────────┐
│  Jaeger        │        │  Prometheus    │  │  Loki           │
│  (traces)          │        │  (metrics)         │  │  (logs)             │
└──────────┘        └──────────┘  └──────────┘
```

**Flow in words:**
1. Your application uses the **OpenTelemetry SDK** to generate spans (traces), metrics, and logs as it runs.
2. This telemetry is sent via **OTLP** (the standard protocol) to the **OpenTelemetry Collector** — a separate, standalone process.
3. The Collector's **receivers** accept the incoming data, **processors** transform/filter/batch it, and **exporters** send it onward to whatever backend(s) you've configured.
4. Because the Collector sits in the middle, you can change WHERE data goes (Jaeger vs Tempo, Prometheus vs Datadog) without touching any application code at all.

---

## OpenTelemetry vs Alternatives

**Visual:**
```
Tool/Approach              Focus                          Notes
─────────────────────────────────────────────────────────────────────
OpenTelemetry                 Vendor-neutral standard           CNCF project, broad language
                                                              support, the industry direction
Datadog/New Relic SDKs           Proprietary instrumentation         Locked to that vendor, though
                                                              many now also SUPPORT OTel input
Jaeger client libraries              Tracing-only, proprietary            Largely superseded by OTel;
                             API (legacy)                          Jaeger now consumes OTLP directly
Zipkin                                 Tracing-only, older standard          Largely superseded by OTel

OpenTelemetry's niche: it isn't a BACKEND itself —
it's the instrumentation and pipeline STANDARD that
feeds into whatever backend you choose (including
Jaeger and Prometheus, which you already have notes
on) — making it complementary to, not competing with,
your existing observability stack.
```

---

## Real-Life DevOps Use Case

**Scenario:** A company has instrumented its 30 microservices using a mix of vendor-specific APM agents (some using Datadog's proprietary tracer, others with no tracing at all) and is now facing a costly Datadog contract renewal, wanting the option to switch to a self-hosted alternative without a risky big-bang migration.

**Without OpenTelemetry:**
- Switching away from Datadog means re-instrumenting every one of the 30 services with a different vendor's proprietary SDK — a multi-month project touching every codebase.
- Services with no tracing at all have zero visibility into cross-service request flow.
- No consistent way to correlate a trace with the logs generated during that same request.

**What the DevOps engineer does:**
1. Introduces the **OpenTelemetry SDK** into each service (starting with auto-instrumentation where available, covered in file 03), replacing proprietary tracer calls with OTel's vendor-neutral API.
2. Deploys an **OpenTelemetry Collector** configured to export to BOTH Datadog (keeping existing dashboards working during transition) AND a self-hosted Jaeger + Prometheus + Loki stack simultaneously.
3. Runs both backends in parallel for several weeks, validating the self-hosted stack shows equivalent data before making a decision.
4. When the Datadog contract comes up for renewal, simply **removes the Datadog exporter** from the Collector's configuration — zero application code changes required, since the SDK was never Datadog-specific to begin with.
5. Result: full backend migration achieved by changing a handful of lines in the Collector's config file, rather than re-instrumenting 30 services.

**Result:** OpenTelemetry converts a multi-month, high-risk vendor migration into a low-risk configuration change — precisely the flexibility it was designed to provide.

---

Next: **01installation_and_setup.md** — installing the OpenTelemetry SDK and Collector, and instrumenting your first application.