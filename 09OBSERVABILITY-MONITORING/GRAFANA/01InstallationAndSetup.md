# Grafana - Installation & Setup

Getting Grafana running, logging in for the first time, and connecting your first data source.

## Table of Contents
- [Installation Options](#installation-options)
- [Docker Installation](#docker-installation)
- [Docker Compose with Prometheus](#docker-compose-with-prometheus)
- [First Login](#first-login)
- [Grafana UI Layout](#grafana-ui-layout)
- [Adding a Data Source](#adding-a-data-source)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installation Options

**Visual:**
```
┌──────────────────────────────────────────────────┐
│ Method                Best For                        │
├──────────────────────────────────────────────────┤
│ Docker                 Quickest local setup, CI/testing  │
│ Binary/Package (apt/yum) Traditional VM-based production    │
│ Helm Chart (Kubernetes)   Production Kubernetes deployments  │
│ Grafana Cloud (SaaS)       Hosted, no infra to manage at all │
└──────────────────────────────────────────────────┘
```

---

## Docker Installation

```bash
docker run -d --name=grafana \
  -p 3000:3000 \
  grafana/grafana-oss
```

**Visual:**
```
Container Startup:
┌────────────────────────────────┐
│  Container: grafana                 │
│  ┌──────────────────────────┐    │
│  │  Web Server: port 3000       │    │
│  │  Embedded SQLite DB (default)  │    │  ← dashboards/users stored here
│  └──────────────────────────┘    │
└────────────────────────────────┘
         ↓ exposed on
   http://localhost:3000
```

⚠️ **Production note:** like the embedded SQLite database, this is fine for evaluation. Production setups typically point Grafana at an external Postgres/MySQL database for its own config storage, so container restarts don't risk config loss:

```bash
docker run -d --name=grafana \
  -p 3000:3000 \
  -e GF_DATABASE_TYPE=postgres \
  -e GF_DATABASE_HOST=postgres-host:5432 \
  -e GF_DATABASE_NAME=grafana \
  -e GF_DATABASE_USER=grafana \
  -e GF_DATABASE_PASSWORD=grafana_password \
  grafana/grafana-oss
```

---

## Docker Compose with Prometheus

A realistic local setup pairs Grafana with a data source immediately.

```yaml
version: "3"
services:
  grafana:
    image: grafana/grafana-oss
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

volumes:
  grafana_data:
```

```bash
docker-compose up -d
```

**Visual:**
```
Stack Relationship:
┌─────────────┐   scrapes   ┌──────────────┐
│  Prometheus     │ ───────────→ │  Target apps    │
│  (port 9090)      │               │  (/metrics)       │
└─────────────┘               └──────────────┘
       ↑
       │ queried by
┌─────────────┐
│  Grafana        │
│  (port 3000)      │
└─────────────┘
```

---

## First Login

Navigate to `http://localhost:3000`

**Default credentials:** `admin` / `admin` (forced password change on first login)

**Visual:**
```
Login Screen                Home Dashboard (after login)
┌───────────────┐          ┌─────────────────────────────┐
│ Username: admin│          │  Welcome to Grafana              │
│ Password: admin│ ───────→ │  [+ Create Dashboard]              │
│  [Log In]       │          │  Connections  Dashboards  Alerting  │
└───────────────┘          └─────────────────────────────┘
```

---

## Grafana UI Layout

**Visual:**
```
┌────────────────────────────────────────────────────────┐
│ ☰   Search   [+]                              🔔  ⚙️  👤  │
├──────────┬─────────────────────────────────────────────┤
│  Home       │                                                │
│  Dashboards  │             Main Content Area                     │
│  Explore      │        (dashboard grid, panel editor,               │
│  Alerting      │         data source config, etc. —                   │
│  Connections    │         changes based on left nav selection)          │
│  Administration  │                                                        │
│  (Left Nav)       │                                                          │
└──────────┴─────────────────────────────────────────────┘
```

**Key sections:**
```
Dashboards       → browse/create saved dashboards
Explore          → ad-hoc querying without building a full dashboard
                   (great for quick investigation during incidents)
Alerting         → alert rules, contact points, notification policies
Connections      → data sources and plugins
Administration   → users, teams, organizations, permissions
```

---

## Adding a Data Source

```
Connections → Data Sources → Add data source → Prometheus
```

**Configure:**
```
Name: Prometheus
URL: http://prometheus:9090
Access: Server (default)
```

**Visual:**
```
Data Source Config Screen:
┌─────────────────────────────────────┐
│  HTTP                                    │
│  URL: http://prometheus:9090               │
│                                             │
│  Auth (if needed)                            │
│  ☐ Basic Auth   ☐ TLS Client Auth              │
│                                             │
│  [Save & Test]                                │
└─────────────────────────────────────┘
         ↓
  ✓ "Data source is working" (green success message)
```

**Visual — the "Access: Server" setting explained:**
```
Server (default, recommended):
Browser → Grafana Server → Data Source
(Grafana backend proxies the request — data source
 doesn't need to be reachable from users' browsers,
 only from the Grafana server itself)

Browser (legacy, rarely used now):
Browser → Data Source directly
(requires data source to be reachable from every
 user's browser — CORS issues, less secure, avoid)
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up observability for a new Kubernetes cluster and needs Grafana connected to multiple data sources from day one.

**What they do:**
1. Deploys Grafana via **Helm chart** into the cluster (rather than a standalone Docker container) so it's managed the same way as other cluster workloads, with persistent storage backed by a PVC instead of ephemeral container storage.
2. Configures the **Prometheus data source** pointing at the in-cluster Prometheus service (`http://prometheus.monitoring.svc:9090`), using the "Server" access mode so it queries through the internal cluster network rather than requiring external exposure.
3. Adds a **Loki data source** for centralized logs, and a **CloudWatch data source** for AWS-managed services (RDS, ELB) that live outside the cluster.
4. Immediately sets Grafana's own database to an **external managed Postgres instance** rather than the embedded SQLite default, so dashboard definitions and alert configs survive pod restarts/rescheduling without relying on the pod's own ephemeral storage.
5. Verifies each data source with **Save & Test** before building any dashboards — catching a misconfigured internal DNS name for the Loki service early, rather than debugging "why is my dashboard panel empty" later.

**Why this matters:** Getting the data source and persistence configuration right from the start avoids the common pitfall of building beautiful dashboards on top of a Grafana instance that loses all its configuration on the next pod restart.

---

Next: **02core_concepts_dashboards_panels.md** — the theory behind dashboards, panels, visualizations, and how queries actually work.