# Alertmanager Command Cheat Sheet

Quick reference for all Alertmanager commands and config patterns covered across the guides.

---

## Install & Setup

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install alertmanager prometheus-community/alertmanager -n monitoring --create-namespace
# or full stack:
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring

kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
amtool check-config alertmanager.yml
```

## Config Skeleton

```yaml
global:
  resolve_timeout: 5m
route:
  receiver: 'default'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="critical"]
      receiver: 'pagerduty-oncall'
receivers:
  - name: 'default'
    slack_configs: [{channel: '#alerts'}]
  - name: 'pagerduty-oncall'
    pagerduty_configs: [{service_key: '<key>'}]
inhibit_rules:
  - source_match: {severity: 'critical'}
    target_match: {severity: 'warning'}
    equal: ['alertname', 'namespace']
```

## Routing & Testing

```bash
amtool config routes test --config.file=alertmanager.yml severity=critical team=payments
curl -X POST http://localhost:9093/-/reload
```

## Silences

```bash
amtool silence add alertname="HighCPU" namespace="my-app" \
  --duration="2h" --comment="JIRA-1234" --author="priya"
amtool silence query
amtool silence expire <silence-id>
```

## Test Alert (End-to-End)

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"critical"},
        "annotations":{"summary":"test"},
        "startsAt":"'$(date -u +%Y-%m-%dT%H:%M:%S.000Z)'"}]'

curl http://localhost:9093/api/v2/alerts
curl http://localhost:9093/api/v2/status
```

## HA / Clustering

```bash
alertmanager --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-0.alertmanager-headless:9094 \
  --cluster.peer=alertmanager-1.alertmanager-headless:9094
```

```yaml
alertmanager:
  alertmanagerSpec:
    replicas: 3
```

## Upgrade

```bash
helm upgrade alertmanager prometheus-community/alertmanager -n monitoring --version 1.x.x
```

---

## Command Quick Index

| Command | Purpose |
|---|---|
| `amtool check-config` | Validate config syntax before applying |
| `amtool config routes test` | Dry-run test which receiver an alert routes to |
| `amtool silence add` | Create a temporary silence |
| `amtool silence expire` | End a silence early |
| `curl .../-/reload` | Hot-reload config without restart |
| `curl .../api/v2/alerts` | POST a synthetic alert or list current alerts |
| `curl .../api/v2/status` | Check cluster health and peer list |

---

## Concept Quick Index

| Concept | Purpose |
|---|---|
| `route` | Decision tree for where alerts go |
| `matchers` | Conditions to match alerts by label |
| `continue: true` | Let an alert match multiple routes |
| `group_by` | Batch related alerts into one notification |
| `group_wait / group_interval / repeat_interval` | Notification pacing timers |
| `receivers` | Notification destinations (Slack/Email/PagerDuty/webhook) |
| `inhibit_rules` | Auto-suppress redundant alerts |
| `mute_time_intervals` | Recurring scheduled silencing |
| Gossip protocol | Cross-replica deduplication in HA mode |

---

This cheat sheet summarizes all Alertmanager commands and config patterns from the basics, routing/grouping, receivers/notifications, silencing/inhibition, HA/clustering, and production operations guides.