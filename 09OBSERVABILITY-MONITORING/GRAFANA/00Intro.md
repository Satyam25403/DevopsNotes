# Grafana - Introduction & Architecture

Understanding what Grafana is, where it fits in the observability stack, and why DevOps teams build their monitoring around it.

## Table of Contents
- [What is Grafana](#what-is-grafana)
- [Grafana Does Not Store Data](#grafana-does-not-store-data)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [The Observability Stack](#the-observability-stack)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [Grafana vs Alternatives](#grafana-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is Grafana

**Grafana is an open-source visualization and dashboarding platform** that connects to many different data sources and turns raw metrics, logs, and traces into readable dashboards, graphs, and alerts.

**Visual:**
```
Raw Data (hard to read)                  Grafana Dashboard
┌──────────────────────┐                ┌─────────────────────────┐
│ cpu_usage{host="web1"}  │                │  CPU Usage                 │
│  = 0.87, 0.82, 0.91...    │   ───────→    │  ┌───────────────────┐  │
│                            │                │  │     ╱╲    ╱╲          │  │
│ http_requests_total          │                │  │   ╱    ╲  ╱    ╲      │  │
│  {status="500"} = 1204        │                │  │  ╱      ╲╱      ╲      │  │
│                                 │                │  └───────────────────┘  │
│ (numbers in a database,           │                │  87% ▲ (threshold: 80%)   │
│  meaningless at a glance)            │                └─────────────────────────┘
└──────────────────────┘
```

---

## Grafana Does Not Store Data

**This is the single most important architectural fact about Grafana.**

**Visual:**
```
Common Misconception:
"Grafana stores my metrics/logs"  ❌ WRONG

Reality:
┌──────────┐        query         ┌──────────────────┐
│  Grafana     │  ───────────────────→ │  Data Source          │
│  (visualization│                       │  (Prometheus, Loki,    │
│   layer only)   │  ←───────────────────│   InfluxDB, MySQL,      │
└──────────┘        results          │   Elasticsearch, etc.)   │
                                      └──────────────────┘

Grafana's own database (SQLite/MySQL/Postgres) stores ONLY:
- Dashboard definitions (JSON)
- User accounts, org settings
- Alert rules, notification config
- Data source connection settings

It does NOT store your metrics or logs — those live in
whatever backend system you point it at.
```

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                                How Grafana Helps
──────────────────────────────────────────────────────────────────
Metrics scattered across many tools     Single pane of glass across Prometheus,
(Prometheus, CloudWatch, DBs, logs)     Loki, CloudWatch, and more, in one dashboard
No visibility into system health         Real-time dashboards for CPU, memory,
                                        latency, error rates, business metrics
Finding out about outages from users      Alerting rules notify the team BEFORE
instead of monitoring                    customers notice
Different teams need different views      Dashboards + folders + permissions per team
Correlating metrics with logs during        Explore view lets you jump from a metric
an incident is slow and manual              spike directly to related logs
```

---

## The Observability Stack

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                  The "LGTM" / "PLG" Stack                    │
├─────────────────────────────────────────────────────────┤
│                                                            │
│  Metrics  →  Prometheus / Mimir     →  numbers over time      │
│  Logs     →  Loki                   →  text log lines            │
│  Traces   →  Tempo / Jaeger          →  request-level tracing       │
│                                                            │
│  ALL visualized and correlated through:                      │
│                                                            │
│                     GRAFANA                                    │
│           (the single visualization layer)                       │
└─────────────────────────────────────────────────────────┘
```

**This is why you saw Grafana grouped with Loki and Prometheus in your OBSERVABILITY folder** — the three form a common, complementary open-source observability stack: Prometheus for metrics, Loki for logs, Grafana to visualize and correlate both.

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                     Grafana Concepts                      │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Data Source        →  Connection to where data lives       │
│                        (Prometheus, Loki, MySQL, etc.)         │
│                                                           │
│  Dashboard          →  A collection of panels, saved as JSON  │
│                                                           │
│  Panel              →  A single visualization (graph, table,   │
│                        gauge, stat, etc.)                       │
│                                                           │
│  Query              →  The actual request sent to a data        │
│                        source (PromQL, LogQL, SQL, etc.)          │
│                                                           │
│  Variable            →  A dropdown/template value that makes       │
│                        one dashboard reusable across environments   │
│                                                           │
│  Alert Rule           →  A condition that, when breached, fires        │
│                        a notification                                  │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
                    ┌─────────────────────────┐
                    │        Grafana Server        │
                    │  ┌──────────────────────┐  │
                    │  │  Web UI                  │  │
                    │  ├──────────────────────┤  │
                    │  │  Query Engine               │  │  ← translates panel
                    │  ├──────────────────────┤  │     requests into data
                    │  │  Alerting Engine              │  │     source queries
                    │  ├──────────────────────┤  │
                    │  │  Grafana's own DB               │  │  ← dashboards, users,
                    │  │  (SQLite/MySQL/Postgres)           │  │     alert config only
                    │  └──────────────────────┘  │
                    └────────────┬────────────────┘
                                 │  queries (PromQL/LogQL/SQL/etc)
                    ┌────────────┼────────────┬──────────────┐
                    ↓            ↓            ↓              ↓
              ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐
              │Prometheus │ │  Loki      │ │  MySQL     │  │CloudWatch │
              └──────────┘ └──────────┘ └──────────┘  └──────────┘
```

---

## Grafana vs Alternatives

**Visual:**
```
Tool                Focus                          Notes
─────────────────────────────────────────────────────────────────────
Grafana               Multi-source visualization       Open-source, huge plugin/
                                                        data-source ecosystem
Kibana                Elasticsearch-native visualization Tied closely to the ELK stack
Datadog                All-in-one SaaS (collection +      Commercial, easier turnkey,
                       visualization + alerting)             less flexible/cheaper at scale
New Relic              All-in-one SaaS APM                   Commercial, deep APM focus
CloudWatch Dashboards   AWS-native only                       Simple, but AWS-locked-in

Grafana's niche: connect to almost ANY data source
(including multiple cloud providers at once), fully
open-source core, and a massive community dashboard/plugin library.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer at a company running a mix of on-prem Kubernetes clusters and AWS services needs a single place for the whole engineering org to check system health.

**Without Grafana:**
- Kubernetes metrics live in Prometheus, accessible only via `kubectl port-forward` and raw PromQL queries.
- AWS RDS metrics are only visible in the AWS Console.
- Application logs are scattered across individual server files.
- During an incident, engineers waste the first 15 minutes just figuring out where to look.

**What the DevOps engineer does:**
1. Deploys **Grafana** and connects it to **Prometheus** (Kubernetes cluster metrics), **CloudWatch** (AWS RDS/EC2 metrics), and **Loki** (centralized application logs).
2. Builds a **single "Service Health" dashboard** showing pod CPU/memory (from Prometheus), database connections (from CloudWatch), and recent error logs (from Loki) — all on one screen.
3. Sets up **alert rules** so if error rate crosses 5% or database CPU exceeds 90%, the on-call engineer gets a Slack/PagerDuty notification automatically.
4. During the next incident, an engineer opens the one dashboard, immediately sees the database CPU spike correlates with the error rate increase, and jumps to the exact log lines from that time window — cutting incident diagnosis time from 15 minutes to under 2.

**Result:** Grafana becomes the organization's single pane of glass, replacing "check five different tools" with one dashboard that correlates metrics, logs, and alerts together.

---

Next: **01installation_and_setup.md** — installing Grafana, first login, and connecting your first data source.