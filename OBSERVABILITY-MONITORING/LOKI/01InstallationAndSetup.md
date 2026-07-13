# Loki - Installation & Setup

Getting Loki running, setting up Promtail to ship logs, and querying your first log lines from Grafana.

## Table of Contents
- [Installation Options](#installation-options)
- [Docker Compose: Loki + Promtail + Grafana](#docker-compose-loki--promtail--grafana)
- [Loki Configuration File](#loki-configuration-file)
- [Promtail Configuration File](#promtail-configuration-file)
- [Starting the Stack](#starting-the-stack)
- [Connecting Grafana to Loki](#connecting-grafana-to-loki)
- [Your First Query](#your-first-query)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installation Options

**Visual:**
```
┌──────────────────────────────────────────────────┐
│ Method                Best For                        │
├──────────────────────────────────────────────────┤
│ Docker/Docker Compose   Quick local setup, learning      │
│ Helm Chart (Kubernetes)   Production Kubernetes clusters    │
│ Binary install             Traditional VM-based setups        │
│ Grafana Cloud (SaaS)        Hosted, no infra to manage at all │
└──────────────────────────────────────────────────┘
```

---

## Docker Compose: Loki + Promtail + Grafana

A realistic local setup runs all three together.

```yaml
version: "3"
services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/loki-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - ./promtail-config.yaml:/etc/promtail/promtail-config.yaml
      - /var/log:/var/log
    command: -config.file=/etc/promtail/promtail-config.yaml
    depends_on:
      - loki

  grafana:
    image: grafana/grafana-oss
    ports:
      - "3000:3000"
    depends_on:
      - loki
```

**Visual:**
```
Stack Relationship:
┌──────────┐   reads   ┌──────────┐  pushes  ┌──────────┐
│ /var/log     │ ──────────→│ Promtail     │ ──────────→│  Loki        │
│ (host logs)     │             │ (agent)         │             │ (port 3100)     │
└──────────┘             └──────────┘             └────┬─────┘
                                                          │ queried by
                                                    ┌────┴─────┐
                                                    │  Grafana     │
                                                    │  (port 3000)   │
                                                    └──────────┘
```

---

## Loki Configuration File

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

**Visual:**
```
Key Config Sections:
┌───────────────────────────────────────────┐
│  auth_enabled: false   → single-tenant mode      │
│                          (multi-tenancy in file 05)│
│  storage: filesystem   → simplest local storage,    │
│                          production uses S3/GCS       │
│  schema_config          → defines how the index          │
│                          is structured/versioned            │
└───────────────────────────────────────────┘
```

---

## Promtail Configuration File

**Promtail is the agent responsible for discovering, labeling, and shipping logs to Loki.**

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

**Visual:**
```
Config Breakdown:
┌───────────────────────────────────────────┐
│  positions.yaml    → tracks HOW FAR Promtail    │
│                       has read in each file,        │
│                       so restarts don't re-send        │
│                       or lose logs                       │
│                                                     │
│  clients            → WHERE to push logs (the Loki     │
│                       push API endpoint)                   │
│                                                     │
│  scrape_configs      → WHAT to read and WHAT LABELS       │
│                       to attach (job, __path__ = the         │
│                       file glob pattern to tail)                │
└───────────────────────────────────────────┘
```

---

## Starting the Stack

```bash
docker-compose up -d
```

**Visual:**
```
Startup Sequence:
Loki starts (port 3100) → Promtail starts, connects to Loki,
   begins tailing /var/log/*.log → Grafana starts (port 3000)

Verify Loki is healthy:
curl http://localhost:3100/ready
→ "ready"

Verify Promtail is shipping logs:
curl http://localhost:9080/targets
→ shows the discovered log files being tailed
```

---

## Connecting Grafana to Loki

```
Grafana → Connections → Data Sources → Add data source → Loki
URL: http://loki:3100
[Save & Test]
```

**Visual:**
```
┌─────────────────────────────────────┐
│  Data Source: Loki                       │
│  URL: http://loki:3100                     │
│  [Save & Test]                                │
└─────────────────────────────────────┘
         ↓
  ✓ "Data source connected and labels found"
```

---

## Your First Query

```
Grafana → Explore → select Loki data source
```

```logql
{job="varlogs"}
```

**Visual:**
```
Explore View Output:
┌─────────────────────────────────────────┐
│  Time                    Log line               │
├─────────────────────────────────────────┤
│  10:15:32.001   Jul 12 syslog: started service...│
│  10:15:33.102   Jul 12 syslog: connection accepted│
│  10:15:34.220   Jul 12 syslog: request processed   │
└─────────────────────────────────────────┘

This query says: "show me all log lines from any
stream where the label job equals varlogs" —
the simplest possible LogQL query, just a label selector.
```

**Narrowing with a text filter:**
```logql
{job="varlogs"} |= "error"
```

**Visual:**
```
{job="varlogs"}    → select the stream(s) by label (fast, indexed)
|= "error"           → then filter lines containing "error"
                       (scans only within the already-selected streams)
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up log aggregation for a small Kubernetes cluster and needs it working end-to-end before onboarding the wider team.

**What they do:**
1. Deploys Loki via the **Helm chart** (`grafana/loki-stack`) rather than raw Docker Compose, since the cluster will run in Kubernetes long-term — this chart also conveniently bundles Promtail configured to auto-discover pod logs.
2. Confirms Promtail's **DaemonSet** is running on every node (one Promtail pod per node is the standard pattern, ensuring every node's container logs get tailed regardless of which node a pod is scheduled to).
3. Verifies via `/targets` that Promtail is correctly picking up the expected labels (`namespace`, `pod`, `container`) auto-attached by Kubernetes service discovery — a common early mistake is assuming labels exist when the scrape config actually needs explicit relabeling (covered in file 04).
4. Connects Grafana to Loki and runs a first sanity query, `{namespace="production"}`, confirming logs are flowing before telling the team it's ready to use.
5. Sets `positions.filename` to a **persistent volume** rather than ephemeral pod storage, so a Promtail pod restart doesn't lose track of its read position and either skip or re-send logs.

**Why this matters:** Getting the agent's label discovery and position-tracking right during initial setup avoids the two most common "logs are missing" support tickets later — mislabeled streams and Promtail losing its place after a restart.

---

Next: **02labels_and_architecture_deep_dive.md** — the theory of labels, cardinality, and Loki's internal components in depth.