# Loki - Log Ingestion Pipelines

Practical agent configuration: Kubernetes service discovery, relabeling, multi-source ingestion, and parsing pipelines that shape logs before they reach Loki.

## Table of Contents
- [The Role of the Agent](#the-role-of-the-agent)
- [Promtail on Kubernetes](#promtail-on-kubernetes)
- [Relabeling Deep Dive](#relabeling-deep-dive)
- [Pipeline Stages](#pipeline-stages)
- [Multi-Source Ingestion](#multi-source-ingestion)
- [Grafana Alloy (Promtail's Successor)](#grafana-alloy-promtails-successor)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Role of the Agent

**Visual:**
```
Without proper agent configuration:
Raw log line arrives at Loki with generic/missing labels →
   {filename="/var/log/containers/xyz.log"} →
   basically unusable for filtering by app/team/environment

With properly configured agent:
Raw log line → agent DISCOVERS metadata (which pod, namespace,
   container this came from) → RELABELS into meaningful labels →
   {app="payments", namespace="production", container="api"} →
   Immediately useful for real queries
```

The agent (Promtail, or its successor Grafana Alloy) is where most of the practical "make Loki actually useful" work happens.

---

## Promtail on Kubernetes

**Deployed as a DaemonSet — one Promtail pod per node, tailing that node's container logs.**

```yaml
# promtail-k8s-config.yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
```

**Visual:**
```
kubernetes_sd_configs: role: pod
        ↓
Promtail asks the Kubernetes API: "what pods exist
on this node, and what are their labels/annotations?"
        ↓
For each discovered pod, Kubernetes provides METADATA
labels prefixed with __meta_kubernetes_*
        ↓
relabel_configs SELECTIVELY promotes specific metadata
into actual Loki labels (app, namespace, pod, container)
        ↓
Only the labels you explicitly promote become
INDEXED Loki labels — everything else is discarded
(intentionally, to avoid cardinality explosion, see file 02)
```

---

## Relabeling Deep Dive

**Relabeling is how you control exactly which Kubernetes metadata becomes a Loki label — and it's also how you DROP unwanted logs entirely.**

**Visual:**
```
Common Relabel Actions:
┌─────────────────────────────────────────────────┐
│  replace       → copy a source label to a target label  │
│  keep           → ONLY keep targets matching a regex        │
│  drop            → DISCARD targets matching a regex           │
│  labelmap         → bulk-rename __meta_* labels                  │
└─────────────────────────────────────────────────┘
```

**Example: only ingest logs from the "production" namespace:**
```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    regex: production
    action: keep
```

**Example: drop noisy healthcheck/readiness-probe logs entirely:**
```yaml
pipeline_stages:
  - match:
      selector: '{app="payments"}'
      stages:
        - drop:
            expression: ".*healthz.*"
```

**Visual:**
```
Why dropping matters:
Health check endpoints (e.g. /healthz) often get hit
every few SECONDS by Kubernetes liveness probes —
without dropping these, a huge fraction of ingested
log volume is pure noise, inflating storage costs
and cluttering every query with irrelevant lines.
```

---

## Pipeline Stages

**Pipeline stages process each log line as it's ingested — parsing, extracting labels, dropping, and transforming.**

**Visual:**
```
Log Line Flows Through Pipeline Stages IN ORDER:
Raw line → Stage 1 (parse JSON) → Stage 2 (extract label from
   a field) → Stage 3 (drop if condition matches) → Final line
   + labels sent to Loki
```

**Example: extract log level from JSON and promote it to a label:**
```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        msg: message
  - labels:
      level:
  - output:
      source: msg
```

**Visual:**
```
Raw log line:
{"level":"error","message":"payment declined","user":"4821"}

Stage: json → extracts level=error, msg="payment declined"
Stage: labels → PROMOTES level to an actual Loki label
                (small cardinality: debug/info/warn/error — SAFE)
Stage: output → replaces the stored line content with
                just the "msg" field, discarding the rest
                (reduces storage, keeps only what's useful)

Result stream: {app="payments", level="error"}
Stored line: "payment declined"

Note: "user" was NOT promoted to a label — correctly
avoiding the cardinality trap from file 02.
```

**Example: timestamp parsing (use the log's own timestamp, not ingestion time):**
```yaml
pipeline_stages:
  - json:
      expressions:
        timestamp: timestamp
  - timestamp:
      source: timestamp
      format: RFC3339
```

**Visual:**
```
Why this matters:
Without explicit timestamp parsing, Loki uses the
TIME IT RECEIVED the log line as its timestamp —
if there's any delay in shipping (network issues,
backpressure), logs can appear "out of order" or
clustered at the wrong time in queries.

Parsing the log's OWN embedded timestamp keeps
query time ranges accurate to when the event
actually happened in the application.
```

---

## Multi-Source Ingestion

**Real organizations often need logs from more than just Kubernetes pods.**

**Visual:**
```
┌─────────────────────────────────────────────────┐
│ Source                    Scrape Config Type          │
├─────────────────────────────────────────────────┤
│ Kubernetes pods              kubernetes_sd_configs          │
│ Plain VM/server log files       static_configs + __path__      │
│ Systemd journal                    journal (Promtail-specific)     │
│ Docker containers (non-K8s)          docker_sd_configs                │
│ Cloud provider logs (via push)          Loki push API directly,           │
│                                    e.g. from a Lambda function              │
└─────────────────────────────────────────────────┘
```

**Example combining Kubernetes and static file scraping in one Promtail instance:**
```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    # ... relabel_configs as before

  - job_name: nginx-access-logs
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          __path__: /var/log/nginx/access.log
```

---

## Grafana Alloy (Promtail's Successor)

**Grafana Alloy is the newer, unified agent — replacing Promtail (logs), and also handling metrics/traces in one binary.**

**Visual:**
```
Old World:
Promtail (logs only) + Prometheus node_exporter/agent (metrics)
+ separate tracing agent = THREE separate agents to manage

New World (Alloy):
┌───────────────────────────────┐
│           Grafana Alloy             │
│  (ONE agent, configurable pipelines,  │
│   handles logs + metrics + traces)      │
└───────────────────────────────┘

Promtail is now in LEGACY/maintenance mode — Grafana Labs
recommends new deployments use Alloy, though Promtail
configs remain widely documented and still functional.
```

**Alloy config concept (River syntax, different from Promtail's YAML):**
```hcl
loki.source.kubernetes "pods" {
  targets = discovery.kubernetes.pods.targets
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

**Visual:**
```
Why this matters for DevOps teams:
New Loki deployments should generally start with Alloy
rather than Promtail — but understanding Promtail's
concepts (scrape configs, relabeling, pipeline stages)
transfers directly, since Alloy implements the same
underlying ideas with a different configuration syntax.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is rolling out log aggregation across a 50-node Kubernetes cluster running 30 different microservices, and needs the ingestion pipeline to be both cost-efficient and immediately useful for each team.

**What they configure:**
1. Deploys Promtail as a **DaemonSet** with Kubernetes service discovery, relabeling `__meta_kubernetes_pod_label_app`, `__meta_kubernetes_namespace`, and `__meta_kubernetes_pod_container_name` into clean `app`, `namespace`, and `container` labels — giving every team immediately filterable logs without any extra work on their part.
2. Adds a **drop pipeline stage** filtering out lines matching `/healthz` and `/readyz` paths — after checking ingestion volume, discovers this alone reduces total log volume by around 30%, directly lowering storage costs.
3. Adds a **JSON parsing pipeline stage** that promotes only the `level` field (low cardinality: debug/info/warn/error) to a label, while explicitly leaving fields like `user_id` and `request_id` out of the label set — preventing the cardinality explosion problem from file 02 before it can ever happen.
4. Configures **explicit timestamp parsing** from each service's JSON `timestamp` field, since several services run in different timezones and without this, log ordering during cross-service incident correlation was subtly wrong.
5. For a handful of **legacy VMs not yet migrated to Kubernetes**, runs a separate Promtail instance with `static_configs` tailing their log files directly, feeding into the same central Loki instance — giving the team one unified log view across both Kubernetes and legacy infrastructure during the migration period.

**Why this matters:** The ingestion pipeline is where most of the real engineering judgment in a Loki deployment happens — get the relabeling and cardinality decisions right here, and querying/dashboards/alerting downstream (files 03 and Grafana's alerting) become simple and cheap; get it wrong, and no amount of clever LogQL fixes an overloaded, cardinality-exploded cluster.

---

Next: **05advanced_realworld_usecases.md** — retention policies, storage backends at scale, multi-tenancy, and mature real-world Loki operating practices.