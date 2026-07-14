# Alertmanager Basics - What It Is, Architecture, Install

Essential Alertmanager concepts and setup for handling alerts fired by Prometheus, with visual diagrams.

## Table of Contents
- [What is Alertmanager](#what-is-alertmanager)
- [Where Alertmanager Sits in the Pipeline](#where-alertmanager-sits-in-the-pipeline)
- [Architecture](#architecture)
- [Install via Helm](#install-via-helm)
- [The Config File Structure](#the-config-file-structure)
- [Verify Installation](#verify-installation)
- [Your First Alert Flow](#your-first-alert-flow)

---

## What is Alertmanager

**Alertmanager handles alerts sent by client applications like Prometheus. It deduplicates, groups, routes, silences, and dispatches them to the right notification channel (Slack, email, PagerDuty, etc.) - Prometheus itself does NOT send notifications directly.**

**Visual:**
```
Without Alertmanager:
Prometheus fires the same alert 500 times (once per pod)
┌────────────────────────────────────┐
│ 500 separate Slack messages spammed   │
│ On-call misses the important one         │
└────────────────────────────────────┘

With Alertmanager:
Prometheus fires the same alert 500 times
        │
        ▼
┌────────────────────────────────────┐
│ Alertmanager groups them into ONE       │
│ notification: "500 pods are down"          │
│ Routes it to the right team's channel         │
│ Silences it if already being worked on          │
└────────────────────────────────────────┘
```

**What it handles:**
- Grouping - combine similar alerts into a single notification
- Deduplication - collapse repeated firing of the same alert
- Routing - send different alerts to different teams/channels based on labels
- Silencing - temporarily mute alerts (planned maintenance, known issues)
- Inhibition - suppress an alert if a more severe, related alert is already firing

---

## Where Alertmanager Sits in the Pipeline

**Visual:**
```
┌──────────────┐   scrapes    ┌──────────────┐
│  Application    │ ───────────→ │  Prometheus     │
│  /metrics          │              │                    │
└──────────────┘              └───────┬──────┘
                                        │ evaluates alerting
                                        │ rules, fires alerts
                                        ▼
                               ┌──────────────┐
                               │ Alertmanager     │
                               │ groups, routes,     │
                               │ dedupes, silences      │
                               └───────┬──────┘
                                        │ dispatches
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
              ┌──────────┐        ┌──────────┐        ┌──────────┐
              │  Slack     │        │ PagerDuty  │        │  Email      │
              └──────────┘        └──────────┘        └──────────┘

Prometheus decides WHAT is wrong (via alerting rules)
Alertmanager decides WHO gets told and HOW
```

---

## Architecture

**Visual:**
```
┌─────────────────────────────────────────────────┐
│              Alertmanager Pod(s)                     │
│                                                        │
│  ┌────────────────────────────────────────┐         │
│  │ Alert Store (in-memory + optional         │          │
│  │  persistent notification log)                │          │
│  └────────────────────────────────────────┘         │
│                                                        │
│  Config: alertmanager.yml                                │
│  ┌────────────────────────────────────────┐             │
│  │ route:      (routing tree)                   │             │
│  │ receivers:  (notification destinations)         │             │
│  │ inhibit_rules: (suppression logic)                 │             │
│  │ templates:  (custom message formatting)              │             │
│  └────────────────────────────────────────┘             │
└─────────────────────────────────────────────────┘
              ▲                              │
              │ POST /api/v2/alerts             │ notify
              │                              ▼
      ┌──────────────┐              ┌──────────────────┐
      │  Prometheus     │              │ Slack/Email/PagerDuty│
      └──────────────┘              └──────────────────┘
```

**Key concepts:**
| Concept | Role |
|---|---|
| `route` | Tree of rules deciding WHICH receiver an alert goes to |
| `receiver` | A named notification destination (Slack channel, email, webhook, etc.) |
| `inhibit_rules` | Suppress lower-priority alerts when a related higher-priority one fires |
| `silences` | Temporary, manually-created mutes for matching alerts |
| `templates` | Custom Go templates controlling notification message formatting |

---

## Install via Helm

**Almost always installed alongside Prometheus via the kube-prometheus-stack chart, but can be standalone too.**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install alertmanager prometheus-community/alertmanager -n monitoring --create-namespace
```

**As part of the full stack (common production pattern):**
```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

**Visual:**
```
Namespace: monitoring
┌─────────────────────────────────┐
│ alertmanager-0 (StatefulSet)        │
│ prometheus-0 (StatefulSet)             │
│ grafana (Deployment)                       │
└─────────────────────────────────┘

kube-prometheus-stack wires Prometheus's alerting
config to point at Alertmanager automatically -
saves you from manually connecting the two.
```

---

## The Config File Structure

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

route:
  receiver: 'default-slack'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'

receivers:
  - name: 'default-slack'
    slack_configs:
      - channel: '#alerts'
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - service_key: '<key>'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']
```

**Visual:**
```
alertmanager.yml sections:
global      → defaults applied everywhere (API URLs, timeouts)
route        → the routing tree (see 01routing_and_grouping.md)
receivers      → notification destinations (see 02receivers_and_notifications.md)
inhibit_rules    → suppression logic (see 03silencing_and_inhibition.md)
templates          → custom message formatting
```

---

## Verify Installation

```bash
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
```

**Visual:**
```
Browser: http://localhost:9093
┌────────────────────────────────────┐
│ Alertmanager                          │
│  Alerts tab   - currently firing alerts  │
│  Silences tab  - active/expired silences   │
│  Status tab     - config + cluster health     │
└────────────────────────────────────┘
```

### Validate config syntax before applying

```bash
amtool check-config alertmanager.yml
```

**Visual:**
```
✓ SUCCESS
Found:
 - global config
 - route
 - 3 receivers
 - 1 inhibit rule

Always run this BEFORE applying a config change -
a syntax error in production means NO alerts get
routed anywhere until it's fixed.
```

---

## Your First Alert Flow

**Visual:**
```
1. Prometheus rule fires:
   ALERT HighErrorRate: error_rate > 0.05 for 5m

2. Prometheus POSTs the alert to Alertmanager:
   POST /api/v2/alerts
   [{"labels": {"alertname": "HighErrorRate",
                 "severity": "critical",
                 "namespace": "my-app"}}]

3. Alertmanager evaluates the route tree:
   severity: critical → matches "pagerduty-oncall" receiver

4. Alertmanager groups + waits (group_wait: 30s)
   in case related alerts arrive together

5. Notification sent:
   PagerDuty incident created, on-call paged
```

---

## Visual Summary

```
1. Install
   helm install alertmanager prometheus-community/alertmanager -n monitoring

2. Configure
   alertmanager.yml: route + receivers + inhibit_rules

3. Validate
   amtool check-config alertmanager.yml

4. Verify
   kubectl port-forward svc/alertmanager 9093:9093

5. Connect Prometheus
   prometheus.yml: alerting.alertmanagers pointing at this service

6. Test
   Trigger a test alert, confirm it reaches the right channel
```

**Core Idea:**
```
┌────────────────┐  fires    ┌────────────────┐  routes   ┌────────────────┐
│  Prometheus     │ ────────→ │  Alertmanager   │ ────────→ │  Human/Team     │
│  (detects WHAT)  │           │ (decides WHO/HOW)│           │  (gets notified) │
└────────────────┘           └────────────────┘           └────────────────┘
```

---

This guide covers Alertmanager basics: what it does, where it fits in the Prometheus pipeline, its architecture, and installing/validating your first config, with visual representations of each step.