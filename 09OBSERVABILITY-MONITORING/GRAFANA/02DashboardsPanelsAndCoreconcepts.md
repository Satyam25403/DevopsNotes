# Grafana - Dashboards, Panels & Core Concepts

The core theory behind how dashboards are structured and how Grafana turns queries into visualizations.

## Table of Contents
- [Dashboards](#dashboards)
- [Panels](#panels)
- [Panel Types](#panel-types)
- [The Query Editor](#the-query-editor)
- [Field Options & Thresholds](#field-options--thresholds)
- [Rows and Layout](#rows-and-layout)
- [Time Range Control](#time-range-control)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Dashboards

**A Dashboard is a collection of Panels, saved together as a single JSON document.**

**Visual:**
```
Dashboard: "API Service Health"
┌───────────────────────────────────────────────┐
│  [Request Rate]     [Error Rate]     [P99 Latency]│
│    graph               graph              stat        │
│                                                    │
│  [CPU Usage]          [Memory Usage]                    │
│    gauge                 gauge                           │
│                                                          │
│  [Recent Error Logs]                                       │
│    logs panel (full width)                                   │
└───────────────────────────────────────────────┘

Under the hood, this whole layout is just JSON:
{
  "title": "API Service Health",
  "panels": [ {...}, {...}, {...} ],
  "time": { "from": "now-6h", "to": "now" }
}
```

**Why JSON matters:** dashboards can be version-controlled in Git, generated programmatically, and provisioned automatically — covered in depth in file 05.

---

## Panels

**A Panel is a single visualization tied to one or more queries.**

**Visual:**
```
Panel Anatomy:
┌─────────────────────────────────┐
│  Title: "Request Rate"              │
│  ┌───────────────────────────┐  │
│  │                                 │  │
│  │        (the visualization)        │  │
│  │                                 │  │
│  └───────────────────────────┘  │
│  Query: rate(http_requests_total[5m])│
│  Data source: Prometheus              │
└─────────────────────────────────┘

Every panel = Query + Visualization Type + Display Options
```

**Creating a panel:**
```
Dashboard → Add → Visualization → Select data source →
   Write query → Choose visualization type → Save
```

---

## Panel Types

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Panel Type       Best For                                │
├─────────────────────────────────────────────────────┤
│ Time series        Metrics changing over time (CPU,         │
│                    request rate) — the most common type       │
│                                                          │
│ Stat               A single current number (e.g. "42 active   │
│                    users right now") with optional sparkline    │
│                                                          │
│ Gauge               A value against a range, visually shows       │
│                    "how close to the danger threshold"              │
│                                                          │
│ Bar chart            Comparing discrete categories                    │
│                    (requests per endpoint)                             │
│                                                          │
│ Table                Raw tabular data, sortable/filterable                │
│                                                          │
│ Logs                  Raw log lines (from Loki/Elasticsearch)                │
│                                                          │
│ Heatmap                 Distribution over time (e.g. latency                  │
│                       percentile buckets)                                       │
│                                                          │
│ Node graph               Service dependency/topology visualization              │
└─────────────────────────────────────────────────────┘
```

**Visual — choosing the right panel type:**
```
Question: "What's my current error rate RIGHT NOW?"
→ Stat panel (single big number)

Question: "How has error rate trended over the last 24 hours?"
→ Time series panel (line graph)

Question: "Which of my 12 microservices has the most errors?"
→ Bar chart panel (comparison across categories)

Question: "Is my disk usage approaching the danger zone?"
→ Gauge panel (visual threshold indicator)
```

---

## The Query Editor

**Every panel's data comes from a query, written in whatever language the data source uses.**

**Visual:**
```
┌───────────────────────────────────────────────┐
│ Data Source     Query Language                      │
├───────────────────────────────────────────────┤
│ Prometheus         PromQL                                │
│ Loki                LogQL                                 │
│ MySQL/Postgres        SQL                                    │
│ Elasticsearch          Lucene / DSL                             │
│ CloudWatch               CloudWatch Metrics Insights              │
└───────────────────────────────────────────────┘
```

**Example PromQL query in a panel:**
```
rate(http_requests_total{status="500"}[5m])
```

**Visual — what this query means:**
```
http_requests_total{status="500"}   → the raw counter metric,
                                        filtered to only 500 errors
rate(...[5m])                        → per-second average rate of
                                        increase, over a 5-minute window

Result: a smooth line showing "500 errors per second,
averaged over rolling 5-minute windows" — much more
readable than a raw ever-increasing counter.
```

**Example LogQL query (Loki):**
```
{app="payments"} |= "ERROR" | json | line_format "{{.message}}"
```

**Visual:**
```
{app="payments"}      → select log streams labeled app=payments
|= "ERROR"             → filter to lines containing "ERROR"
| json                  → parse the line as JSON
| line_format ...         → reformat the displayed output
```

---

## Field Options & Thresholds

**Visual:**
```
Threshold Configuration:
┌─────────────────────────────────┐
│  Value              Color            │
│  Base               Green              │
│  70                 Yellow              │
│  90                 Red                  │
└─────────────────────────────────┘

Applied to a Gauge panel showing CPU at 92%:
┌───────────────────┐
│      🔴 92%              │  ← automatically red because
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░  │     it crossed the 90 threshold
└───────────────────┘

This is what makes a dashboard scannable at a glance —
color immediately communicates "this needs attention"
without reading exact numbers.
```

**Unit formatting:**
```
Raw value: 8589934592
Field Unit: bytes → displays as "8 GB" automatically

Raw value: 0.923
Field Unit: percent (0.0-1.0) → displays as "92.3%"
```

---

## Rows and Layout

**Rows group related panels and can be collapsed — useful for large dashboards.**

**Visual:**
```
Dashboard: "Full Platform Overview"
┌───────────────────────────────────────┐
│ ▼ Row: API Services                        │
│    [Request Rate] [Error Rate] [Latency]      │
├───────────────────────────────────────┤
│ ▼ Row: Database                              │
│    [Connections] [Query Time] [Replication]     │
├───────────────────────────────────────┤
│ ▶ Row: Infrastructure (collapsed)               │
└───────────────────────────────────────┘

Collapsing rows lets a single dashboard serve BOTH
a quick-glance summary AND deep-dive detail, depending
on which rows a viewer expands.
```

---

## Time Range Control

**Every dashboard has a global time range affecting all panels simultaneously.**

**Visual:**
```
┌────────────────────────────────────────┐
│  [Last 6 hours ▼]      [🔄 Refresh: 30s ▼] │
└────────────────────────────────────────┘

Changing to "Last 7 days" instantly re-queries
EVERY panel on the dashboard for that new range —
panels don't need individual time configuration
unless specifically overridden per-panel.

Auto-refresh (e.g. every 30 seconds) is essential
for live "situation room" / NOC-style dashboards
during an active incident.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is building the primary on-call dashboard for a checkout service, meant to be the first thing an engineer opens during an incident.

**What they design:**
1. Top row: **Stat panels** for the 3 numbers that matter most at a glance — current error rate, current P99 latency, current requests/second — using **thresholds** (green/yellow/red) so an on-call engineer can tell severity without reading exact numbers half-asleep at 3 AM.
2. Second row: **Time series panels** showing the last 6 hours of the same metrics, so the engineer can immediately see if this is a sudden spike or a gradual degradation — these look very different and suggest different root causes.
3. A **Logs panel** (Loki) filtered to `{app="checkout"} |= "ERROR"`, positioned directly below the metrics panels, so the engineer can correlate a metric spike with the actual error messages without switching to a different tool.
4. Sets the dashboard's **default time range to "Last 3 hours"** with **30-second auto-refresh**, since this dashboard is specifically for active incident response, not historical analysis.
5. Groups less-urgent infrastructure metrics (disk space, node counts) into a **collapsed row** at the bottom — available if needed, but not cluttering the primary view during a time-sensitive incident.

**Why this matters:** A dashboard that shows 40 panels of equal visual weight is nearly useless during a 3 AM page — the design priority is surfacing the 3-5 metrics that answer "how bad is it and what's it related to" immediately, with everything else available but out of the way.

---

Next: **03building_dashboards_practical.md** — hands-on: variables/templating, building a reusable multi-environment dashboard, and practical PromQL/LogQL patterns.