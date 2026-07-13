# Loki - Advanced Features & Real-World Use Cases

Going beyond a single-node setup: retention policies, storage at scale, multi-tenancy, scaling modes, and how mature organizations operate Loki in production.

## Table of Contents
- [Retention Policies](#retention-policies)
- [Storage Backends at Scale](#storage-backends-at-scale)
- [Multi-Tenancy](#multi-tenancy)
- [Deployment Modes: Monolithic vs Simple Scalable vs Microservices](#deployment-modes-monolithic-vs-simple-scalable-vs-microservices)
- [Caching for Query Performance](#caching-for-query-performance)
- [Ruler: Recording and Alert Rules on Logs](#ruler-recording-and-alert-rules-on-logs)
- [Common Pitfalls & War Stories](#common-pitfalls--war-stories)
- [Real-Life DevOps Use Case (End-to-End)](#real-life-devops-use-case-end-to-end)

---

## Retention Policies

**Logs shouldn't be kept forever — retention controls how long data is stored before automatic deletion.**

**Visual:**
```
Compactor Component:
Runs periodically → checks chunk age against retention config →
   deletes chunks older than the configured period
```

**Configuration:**
```yaml
limits_config:
  retention_period: 720h  # 30 days

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  retention_delete_delay: 2h
```

**Per-tenant retention override (different teams, different needs):**
```yaml
limits_config:
  retention_period: 720h  # default: 30 days

overrides:
  payments-team:
    retention_period: 2160h  # 90 days, for compliance reasons
  internal-tools-team:
    retention_period: 168h   # 7 days, low value logs
```

**Visual:**
```
Why per-tenant retention matters:
┌─────────────────────────────────────────┐
│  Team              Retention        Reason      │
├─────────────────────────────────────────┤
│  Payments             90 days          Compliance   │
│  Standard services       30 days          Default          │
│  Internal tools            7 days           Low value,           │
│                                          save storage cost           │
└─────────────────────────────────────────┘
Storing everything at the STRICTEST retention
requirement wastes money; storing everything at
the LOOSEST requirement risks compliance violations.
Per-tenant overrides let you tune each appropriately.
```

---

## Storage Backends at Scale

**Visual:**
```
Local Filesystem (dev/small setups):
┌──────────────┐
│  Loki container   │
│  local disk chunks  │  ← lost if the container/node dies
└──────────────┘

Object Storage (production):
┌──────────────┐        ┌──────────────────┐
│  Loki (stateless)│ ─────→ │  S3 / GCS / Azure     │
│  ingesters/       │        │  Blob Storage             │
│  queriers            │        │  (durable, cheap,          │
└──────────────┘        │   virtually unlimited)        │
                         └──────────────────┘
```

**S3 configuration:**
```yaml
common:
  storage:
    s3:
      endpoint: s3.us-east-1.amazonaws.com
      bucketnames: my-loki-chunks
      region: us-east-1
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}
```

**Visual:**
```
Why object storage matters at scale:
- Loki's ingesters/queriers become STATELESS — they can be
  scaled up/down or replaced without losing data, since the
  actual chunks live durably in S3, not on local disk
- S3 storage costs a small fraction of equivalent block storage
  or Elasticsearch-style indexed storage
- Built-in durability/replication handled by the cloud provider,
  not something the Loki operator needs to manage themselves
```

---

## Multi-Tenancy

**Loki natively supports serving multiple isolated "tenants" (teams, customers, business units) from one cluster.**

**Visual:**
```
┌─────────────────────────────────────────────┐
│                  Single Loki Cluster              │
│  ┌──────────────┐  ┌──────────────┐        │
│  │  Tenant: payments │  │  Tenant: platform  │        │
│  │  (isolated data,     │  │  (isolated data,     │        │
│  │   own retention,       │  │   own retention,       │        │
│  │   own rate limits)        │  │   own rate limits)        │        │
│  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────┘

Each tenant's data is completely isolated —
tenant A can never query or see tenant B's logs,
even though they share the same physical infrastructure.
```

**Enabling multi-tenancy:**
```yaml
auth_enabled: true
```

**Sending logs with a tenant ID (Promtail):**
```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
    tenant_id: payments-team
```

**Querying as a specific tenant:**
```bash
curl -H "X-Scope-OrgID: payments-team" \
  "http://loki:3100/loki/api/v1/query?query={app=\"payments\"}"
```

**Visual — why multi-tenancy matters:**
```
Without it:
One Loki cluster per team → massive operational overhead,
   N clusters to patch/upgrade/monitor/scale independently

With multi-tenancy:
ONE Loki cluster, shared infrastructure cost, but each
team's data, rate limits, and retention are fully isolated
via the X-Scope-OrgID header — significant operational
savings for platform teams supporting many internal teams.
```

---

## Deployment Modes: Monolithic vs Simple Scalable vs Microservices

**Visual:**
```
┌──────────────────────────────────────────────────┐
│ Mode                 When to Use                      │
├──────────────────────────────────────────────────┤
│ Monolithic              Small setups, all components       │
│                        run in ONE process — simplest,          │
│                        good for getting started/low volume       │
│                                                          │
│ Simple Scalable           Medium setups — splits into just       │
│                        "read path" and "write path"                │
│                        deployments, each scaled independently         │
│                                                          │
│ Microservices              Large setups — every component            │
│                        (distributor, ingester, querier, etc.)           │
│                        runs as its own independently-scaled                │
│                        deployment, maximum flexibility/scale                   │
└──────────────────────────────────────────────────┘
```

**Visual:**
```
Growth Path:
Monolithic (proof of concept, <10GB/day)
     ↓  (log volume grows)
Simple Scalable (read/write split, tens of GB/day)
     ↓  (log volume grows further)
Microservices (fine-grained scaling per component,
               hundreds of GB/day or more)

Most organizations should START with Simple Scalable
mode directly in production — Monolithic is really best
suited for local testing/dev, and full Microservices mode
is usually only justified at genuinely large scale.
```

---

## Caching for Query Performance

**Visual:**
```
┌─────────────────────────────────────────────┐
│  Query Result Cache      → caches full query results │
│                          for repeated identical            │
│                          queries (e.g. a dashboard              │
│                          refreshing every 30s)                     │
│                                                        │
│  Chunk Cache               → caches decompressed chunk           │
│                          content, avoiding repeated                 │
│                          object storage fetches                        │
│                                                        │
│  Index Cache                 → caches index lookups,                     │
│                          speeding up "which chunks                          │
│                          match these labels" checks                            │
└─────────────────────────────────────────────┘
```

**Configuration (using Memcached):**
```yaml
query_range:
  cache_results: true
  results_cache:
    cache:
      memcached_client:
        addresses: memcached:11211
```

**Visual — why caching matters:**
```
Without caching:
Every dashboard refresh (every 30s) re-executes the
FULL query, re-fetching chunks from S3 every time —
slow, and unnecessarily expensive in S3 request costs

With caching:
Repeated identical queries (extremely common with
auto-refreshing dashboards) hit the cache instead —
dramatically faster, and reduces load on object storage
```

---

## Ruler: Recording and Alert Rules on Logs

**The Ruler component lets Loki evaluate LogQL metric queries on a schedule — powering Grafana-independent alerting and pre-computed recording rules.**

```yaml
# alert-rules.yaml
groups:
  - name: payments-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({app="payments"} | json | level="error" [5m])) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payments error rate is high"
```

**Visual:**
```
Ruler evaluates this LogQL expression on a schedule,
completely independent of whether anyone has Grafana
open — if the condition holds for 5 minutes, it fires
a notification through Alertmanager, the same system
Prometheus alerts use.

This means Loki-based log alerts and Prometheus-based
metric alerts can share ONE unified alerting pipeline
(Alertmanager) rather than needing separate systems.
```

---

## Common Pitfalls & War Stories

**Visual:**
```
Pitfall 1: "Ingesters keep OOMing"
Cause: Cardinality explosion (see file 02) — too many
       unique label combinations held in memory at once
Fix: Audit labels, move high-cardinality fields to
     Structured Metadata or log content instead

Pitfall 2: "Queries covering a wide time range are extremely slow"
Cause: No caching configured, or querying with very broad
       label selectors that match too many streams
Fix: Enable query result + chunk caching; narrow label
     selectors to be as specific as possible

Pitfall 3: "Lost logs after a Promtail pod restart"
Cause: positions.yaml stored on ephemeral pod storage
Fix: Mount positions file on a persistent volume, or
     accept minor gaps as an acceptable tradeoff for
     ephemeral/stateless nodes

Pitfall 4: "One team's log spike drowns out everyone else's alerting/query performance"
Cause: No multi-tenancy or per-tenant rate limits configured
Fix: Enable multi-tenancy with per-tenant ingestion rate
     limits, isolating noisy-neighbor impact

Pitfall 5: "Storage bill higher than expected despite 'cheap' object storage"
Cause: No retention policy configured — logs accumulate forever
Fix: Set explicit retention_period, tuned per tenant/team
     based on actual compliance/business need
```

---

## Real-Life DevOps Use Case (End-to-End)

**Scenario:** A platform team supporting 40 internal teams wants Loki to be a shared, cost-efficient, self-service logging platform — not something each team runs independently.

**Full workflow the team builds:**

1. **Simple Scalable deployment mode**, with read and write paths scaled independently — write path scaled up during peak traffic hours, read path scaled based on dashboard/query load, rather than over-provisioning a single monolithic deployment for worst-case combined load.
2. **S3-backed chunk storage** with lifecycle policies on the S3 bucket itself as a backup safety net, in addition to Loki's own compactor-driven retention — defense in depth for cost control.
3. **Multi-tenancy enabled**, with each of the 40 teams getting their own tenant ID, isolated rate limits (preventing one team's logging spike from degrading query performance for everyone else), and tunable per-tenant retention (compliance-sensitive teams get 90 days, low-value internal tooling gets 7 days).
4. **Standardized Alloy/Promtail configuration** shipped as a shared Helm chart across all teams' clusters, with cardinality-safe relabeling baked in by default — individual teams can't accidentally introduce a `user_id` label without deliberately overriding the shared config.
5. **Caching layer** (Memcached-backed query result and chunk caches) significantly speeding up the platform team's own shared "fleet health" dashboards, which are queried by many viewers simultaneously throughout the day.
6. **Ruler-based alerting** shared with the same Alertmanager instance used for Prometheus metric alerts, giving every team ONE unified place (Slack/PagerDuty routing) for both log-based and metric-based alerts, rather than two disconnected alerting systems.
7. **Quarterly cardinality audits**, using Loki's own internal metrics to catch any team's config drifting toward a cardinality problem before it causes an incident, rather than waiting for ingesters to start OOMing.

**Why this is "real DevOps," not just running a tool:** Loki here isn't just "somewhere logs go" — it's a properly capacity-planned, cost-controlled, multi-tenant platform with governance (cardinality audits, standardized configs) preventing the most common Loki failure modes before they happen, and tightly integrated with the same alerting pipeline as the metrics stack. This is the difference between "we installed Loki" and "Loki is how 40 teams reliably and affordably see their own logs."

---

This completes the Loki note series: **Introduction → Setup → Labels & Architecture → LogQL Practical → Ingestion Pipelines → Advanced/Real-World Usage.**