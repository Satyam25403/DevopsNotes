# Alertmanager High Availability & Clustering - Visual Guide

Running Alertmanager as a resilient, multi-replica cluster so a single pod failure never means missed alerts.

## Table of Contents
- [Why HA Matters for Alertmanager Specifically](#why-ha-matters-for-alertmanager-specifically)
- [The Gossip Protocol](#the-gossip-protocol)
- [Configuring a Cluster](#configuring-a-cluster)
- [How Deduplication Works Across Replicas](#how-deduplication-works-across-replicas)
- [Prometheus Pointing at Multiple Alertmanagers](#prometheus-pointing-at-multiple-alertmanagers)
- [StatefulSet Deployment Pattern](#statefulset-deployment-pattern)
- [Verifying Cluster Health](#verifying-cluster-health)

---

## Why HA Matters for Alertmanager Specifically

**Visual:**
```
Single Alertmanager instance:
┌────────────────────────┐
│ Alertmanager (1 pod)       │
└────────────────────────┘
        │
        ▼
   Pod crashes/node dies
        │
        ▼
┌────────────────────────┐
│ ZERO alerts routed          │
│ ANYWHERE until it's back        │
│ (during the incident that            │
│  probably ALSO caused the pod            │
│  to crash in the first place!)             │
└────────────────────────────────────┘

This is the worst possible failure mode: the alerting
system itself goes down exactly when you need it most
(a real outage often stresses the whole cluster, including
Alertmanager's own node).
```

---

## The Gossip Protocol

**Alertmanager replicas form a peer-to-peer cluster using a gossip protocol to stay in sync about which alerts have already been notified - this is what prevents duplicate pages from multiple replicas.**

**Visual:**
```
┌─────────────┐      gossip      ┌─────────────┐
│ Alertmanager-0 │ ←────────────→ │ Alertmanager-1 │
└─────────────┘                  └─────────────┘
        ▲                                ▲
        │              gossip              │
        └────────────────┬────────────────┘
                          ▼
                  ┌─────────────┐
                  │ Alertmanager-2 │
                  └─────────────┘

Each replica independently receives the SAME alerts
from Prometheus (Prometheus sends to ALL configured
Alertmanagers), but they gossip with each other to
agree "has this notification already been sent?" -
so the human only gets paged ONCE, not 3 times.
```

---

## Configuring a Cluster

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-0.alertmanager-headless:9094 \
  --cluster.peer=alertmanager-1.alertmanager-headless:9094 \
  --cluster.peer=alertmanager-2.alertmanager-headless:9094
```

**Via Helm values (kube-prometheus-stack):**

```yaml
alertmanager:
  alertmanagerSpec:
    replicas: 3
```

**Visual:**
```
--cluster.listen-address    → port used for gossip traffic between peers
--cluster.peer                → address of EACH other replica (repeat per peer)

Helm automates this entirely - setting replicas: 3
generates the correct --cluster.peer flags for a
headless Service automatically, pointing each pod
at its siblings.
```

---

## How Deduplication Works Across Replicas

**Visual:**
```
Prometheus fires an alert
        │
        ├──────────────→ Alertmanager-0  (receives it)
        ├──────────────→ Alertmanager-1  (receives it)
        └──────────────→ Alertmanager-2  (receives it)

All 3 replicas gossip and agree on a deterministic
"who sends the notification" decision (based on the
alert's fingerprint hash and cluster state) -
        │
        ▼
Only ONE notification is actually sent to Slack/PagerDuty,
even though all 3 replicas received and processed the alert

If Alertmanager-0 then crashes:
Alertmanager-1 and Alertmanager-2 already know (via gossip)
what's already been notified - no duplicate, no gap.
```

---

## Prometheus Pointing at Multiple Alertmanagers

**Prometheus itself must be configured to send to ALL Alertmanager replicas, not just one - otherwise you don't actually have HA, just extra idle pods.**

```yaml
# prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager-0.alertmanager-headless:9093'
            - 'alertmanager-1.alertmanager-headless:9093'
            - 'alertmanager-2.alertmanager-headless:9093'
```

**Or via Kubernetes service discovery:**

```yaml
alerting:
  alertmanagers:
    - kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names: ['monitoring']
      relabel_configs:
        - source_labels: [__meta_kubernetes_service_name]
          regex: alertmanager
          action: keep
```

**Visual:**
```
Wrong (defeats the purpose of HA):
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager-0:9093']   ← only ONE target

Right:
alerting:
  alertmanagers:
    - kubernetes_sd_configs: [...]   ← discovers ALL replicas
                                        automatically, survives
                                        replicas being added/removed
```

---

## StatefulSet Deployment Pattern

**Alertmanager is deployed as a StatefulSet (not a Deployment) so each replica gets a stable network identity for gossip peering.**

**Visual:**
```
StatefulSet: alertmanager
┌─────────────────────────────────┐
│ alertmanager-0   (stable DNS name)  │
│ alertmanager-1   (stable DNS name)  │
│ alertmanager-2   (stable DNS name)  │
└─────────────────────────────────┘
            │
            ▼
Headless Service: alertmanager-headless
(provides direct pod-to-pod DNS resolution,
 required for --cluster.peer addresses to work)

A regular Deployment with random pod names would
break gossip peering every time a pod restarts -
StatefulSet's stable naming is what makes clustering
reliable.
```

---

## Verifying Cluster Health

```bash
kubectl port-forward -n monitoring alertmanager-0 9093:9093
curl http://localhost:9093/api/v2/status
```

**Output Example:**
```json
{
  "cluster": {
    "status": "ready",
    "peers": [
      {"name": "01H...", "address": "10.1.2.3:9094"},
      {"name": "01J...", "address": "10.1.2.4:9094"}
    ]
  }
}
```

**Visual:**
```
status: "ready"    → cluster formed successfully, all peers visible
status: "settling"  → still discovering peers, wait a moment
peers: [...]           → should list ALL other replicas; if a peer
                          is missing, check network policies/DNS
                          between Alertmanager pods

Also check the UI: Status tab shows cluster member list
directly in the web interface.
```

---

## Visual Summary

```
1. Why HA               → single instance = total alerting blackout on pod loss
2. Gossip protocol         → replicas agree on "already notified" state
3. --cluster.peer              → wire replicas together (Helm automates this)
4. Deduplication                  → only ONE notification sent despite N replicas
5. Prometheus config                 → MUST list all replicas, not just one
6. StatefulSet + headless Service        → stable identities required for gossip
7. /api/v2/status                           → verify cluster health and peer list
```

---

This guide covers running Alertmanager as a highly-available cluster - gossip-based deduplication, StatefulSet deployment, and correctly wiring Prometheus to all replicas, with visual representations of each mechanism.