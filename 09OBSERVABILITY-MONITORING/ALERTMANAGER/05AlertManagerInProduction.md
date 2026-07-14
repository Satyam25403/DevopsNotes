# Alertmanager in Production - Config Management, Testing, Troubleshooting

Operational patterns for running Alertmanager reliably as the last line of defense before a human gets paged, with visual diagrams.

## Table of Contents
- [Config Reload Without Downtime](#config-reload-without-downtime)
- [GitOps for Alertmanager Config](#gitops-for-alertmanager-config)
- [Testing Alerts End-to-End](#testing-alerts-end-to-end)
- [Alert Fatigue - Designing Good Alerts](#alert-fatigue---designing-good-alerts)
- [Monitoring Alertmanager Itself](#monitoring-alertmanager-itself)
- [Resource Planning](#resource-planning)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Upgrading Alertmanager](#upgrading-alertmanager)

---

## Config Reload Without Downtime

**Alertmanager supports hot-reloading its config via a SIGHUP or HTTP endpoint - no pod restart, no dropped alerts during the reload.**

```bash
curl -X POST http://localhost:9093/-/reload
```

**Or via kubectl (config stored in a ConfigMap/Secret):**

```bash
kubectl create configmap alertmanager-config \
  --from-file=alertmanager.yml -n monitoring --dry-run=client -o yaml \
  | kubectl apply -f -
```

**Visual:**
```
Config change workflow:
┌────────────────────────────────────────┐
│ 1. Edit alertmanager.yml                     │
│ 2. amtool check-config alertmanager.yml         │
│    (validate syntax FIRST)                        │
│ 3. Update ConfigMap/Secret in cluster                │
│ 4. curl -X POST http://.../-/reload                     │
│    (or Prometheus Operator auto-reloads via                │
│     config-reloader sidecar)                                  │
│ 5. Confirm via Status tab: config hash changed                    │
└────────────────────────────────────────┘

kube-prometheus-stack includes a config-reloader
sidecar container that watches the ConfigMap/Secret
and triggers the reload automatically - no manual
curl needed in that setup.
```

---

## GitOps for Alertmanager Config

**Store alertmanager.yml in Git and let ArgoCD/FluxCD sync it - critical config like routing/receivers should never be edited by hand in the cluster.**

**Visual:**
```
Git Repo: monitoring-config/
┌───────────────────────────┐
│ alertmanager.yml               │
│ prometheus-rules.yml               │
└───────────────────────────┘
             │
             ▼ PR reviewed, merged, synced by ArgoCD
┌───────────────────────────┐
│ Cluster: ConfigMap/Secret        │
│  auto-reloaded by sidecar            │
└───────────────────────────┘

Benefit: routing changes go through code review -
"who gets paged for what" is exactly the kind of
change that deserves a second pair of eyes before
it goes live.
```

---

## Testing Alerts End-to-End

**Fire a synthetic test alert directly at Alertmanager's API to confirm the full pipeline (routing → receiver → actual notification) works, without waiting for a real incident.**

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "namespace": "my-app"
    },
    "annotations": {
      "summary": "This is a test alert"
    },
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%S.000Z)'"
  }]'
```

**Visual:**
```
Synthetic Alert Flow:
┌────────────────────────────────────────┐
│ 1. POST synthetic alert to /api/v2/alerts    │
│ 2. Confirm it appears in Alertmanager UI          │
│ 3. Confirm the EXPECTED receiver got the                │
│    real notification (Slack message/PagerDuty page)         │
│ 4. Confirm send_resolved fires when you DELETE                 │
│    or let the test alert expire                                    │
└────────────────────────────────────────┘

Run this after ANY routing/receiver config change -
"the config parsed successfully" is not the same as
"a real notification actually reaches the right person."
```

### Watchdog / dead man's switch pattern

```yaml
- alert: Watchdog
  expr: vector(1)
  labels:
    severity: none
  annotations:
    summary: "This is a permanent, always-firing alert used as a heartbeat"
```

**Visual:**
```
Watchdog alert fires CONSTANTLY (always true)
        │
        ▼
Routed to an external dead-man's-switch service
(e.g. healthchecks.io, Dead Man's Snitch)
        │
        ▼
If Alertmanager/Prometheus itself goes down, the
heartbeat STOPS arriving → external service pages
you about the MONITORING SYSTEM being down, closing
the "who alerts on the alerting system" gap.
```

---

## Alert Fatigue - Designing Good Alerts

**The biggest real-world Alertmanager failure isn't technical misconfiguration - it's poorly designed alerting rules that page people for things that don't need a human.**

**Visual:**
```
Bad Alert Design                     Good Alert Design
┌──────────────────────┐            ┌──────────────────────┐
│ Pages for every            │            │ Pages ONLY for              │
│ transient blip                │            │ sustained, actionable          │
│ (for: 0s)                        │            │ conditions (for: 5m+)              │
│                                    │            │                                        │
│ No runbook_url                       │            │ Always includes runbook_url                │
│ annotation                              │            │ pointing to exact remediation steps            │
│                                            │            │                                                    │
│ Vague summary:                              │            │ Specific summary:                                     │
│ "Something is wrong"                           │            │ "Backend error rate 8% > 5%                              │
│                                                    │            │  threshold for 5 minutes"                                   │
└──────────────────────┘            └──────────────────────┘

Rule of thumb: if an alert fires and nobody needs to
DO anything about it, it shouldn't page - route it to
a low-priority Slack channel or drop it (receiver: 'null')
instead of PagerDuty.
```

---

## Monitoring Alertmanager Itself

```yaml
groups:
  - name: alertmanager-meta-alerts
    rules:
      - alert: AlertmanagerConfigReloadFailed
        expr: alertmanager_config_last_reload_successful == 0
        for: 5m
      - alert: AlertmanagerClusterMemberDown
        expr: alertmanager_cluster_members < 3
        for: 5m
      - alert: AlertmanagerNotificationsFailing
        expr: rate(alertmanager_notifications_failed_total[5m]) > 0
        for: 5m
```

**Visual:**
```
Key metrics to watch:
alertmanager_config_last_reload_successful   → 0 = broken config, silent failure risk
alertmanager_cluster_members                    → should equal replica count
alertmanager_notifications_failed_total            → Slack/PagerDuty API failures
alertmanager_notification_latency_seconds             → slow delivery = late pages

These "meta-alerts" (alerts about the alerting system
itself) should route to a SEPARATE, highly-reliable
channel - ideally the Watchdog/dead-man's-switch pattern
above as a backstop.
```

---

## Resource Planning

**Visual:**
```
Per-replica overhead (typical, small-medium cluster):
┌───────────────────────────────┐
│ CPU request:    100m               │
│ Memory request: 128Mi               │
│ Storage: 1-2Gi (notification log,     │
│  silences, retained for --data.retention)│
└───────────────────────────────┘

--data.retention flag controls how long resolved
alerts/silences are kept in local storage (default 120h) -
increase for longer audit history, at the cost of more
disk usage per replica.
```

---

## Troubleshooting Checklist

```bash
# 1. Is the config valid?
amtool check-config alertmanager.yml

# 2. Is the cluster healthy?
curl http://localhost:9093/api/v2/status

# 3. Which receiver would an alert route to?
amtool config routes test --config.file=alertmanager.yml <labels>

# 4. Are alerts actually arriving from Prometheus?
curl http://localhost:9093/api/v2/alerts

# 5. Check logs for delivery failures
kubectl logs alertmanager-0 -n monitoring
```

**Visual:**
```
Common Issue → Root Cause → Fix
──────────────────────────────────────────────────────────
No notifications at all   → config reload failed silently  → check
                                                                alertmanager_config_
                                                                last_reload_successful

Alert routes to wrong        → matcher typo, or missing        → amtool config
receiver                       continue: true on a sibling       routes test with
                                route                              exact alert labels

Duplicate notifications        → Prometheus only pointed at        → fix Prometheus
from HA setup                    ONE Alertmanager replica              alerting.alertmanagers
                                  instead of all                        to list all replicas

Notification never resolves      → send_resolved: false, or          → set send_resolved:
in Slack                           alert never actually stopped          true, check
                                    firing in Prometheus                   underlying metric

Silence not suppressing            → matcher too narrow (e.g.           → broaden matcher,
anything                              matched a pod name that              use stable labels
                                       no longer exists after restart)       like namespace
```

---

## Upgrading Alertmanager

```bash
helm repo update
helm upgrade alertmanager prometheus-community/alertmanager -n monitoring --version 1.x.x
```

**Visual:**
```
Upgrade Order:
1. Check release notes for config schema changes
   (matchers: syntax, mute_time_intervals are version-gated features)
2. amtool check-config alertmanager.yml
   (confirm current config still validates against new version)
3. helm upgrade (rolling update across StatefulSet replicas,
   one at a time - cluster stays available throughout)
4. Verify cluster re-forms: curl .../api/v2/status
5. Send a synthetic test alert to confirm the full pipeline
```

---

## Visual Summary

```
Production Checklist:
☑ Config changes validated with amtool BEFORE applying
☑ Config stored in Git, synced via GitOps, reviewed via PR
☑ Synthetic test alerts run after every routing/receiver change
☑ Watchdog/dead-man's-switch alert covers "who alerts on the alerter"
☑ Alert design reviewed for fatigue (for: duration, runbook_url, clear summary)
☑ Meta-alerts monitor Alertmanager's own config reload + cluster health
☑ HA cluster verified: correct replica count, Prometheus points at ALL of them
☑ Resource/storage sized for --data.retention window
```

---

This guide covers running Alertmanager operationally in production - safe config changes, end-to-end testing, alert design discipline, self-monitoring, and troubleshooting, with visual representations of each pattern.