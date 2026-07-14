# Alertmanager Receivers & Notifications - Visual Guide

Configuring where alerts actually go - Slack, email, PagerDuty, webhooks - and customizing what the message looks like.

## Table of Contents
- [What is a Receiver](#what-is-a-receiver)
- [Slack Receiver](#slack-receiver)
- [Email Receiver](#email-receiver)
- [PagerDuty Receiver](#pagerduty-receiver)
- [Opsgenie Receiver](#opsgenie-receiver)
- [Generic Webhook Receiver](#generic-webhook-receiver)
- [Custom Notification Templates](#custom-notification-templates)
- [Multiple Receivers per Alert](#multiple-receivers-per-alert)

---

## What is a Receiver

**A named destination that a route points alerts to - one Alertmanager config can define many receivers, each configured for a different channel/tool.**

**Visual:**
```
receivers:
  - name: 'default-slack'      ← just a label used by route.receiver
    slack_configs: [...]
  - name: 'pagerduty-oncall'
    pagerduty_configs: [...]
  - name: 'null'                ← special: silently discards the alert
```

---

## Slack Receiver

```yaml
receivers:
  - name: 'platform-team-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#platform-alerts'
        send_resolved: true
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
```

**Visual:**
```
Slack Message:
┌────────────────────────────────────────┐
│ 🔴 HighErrorRate                          │
│ Backend service error rate above 5%          │
│ [View in Grafana] [Silence]                     │
└────────────────────────────────────────┘

send_resolved: true
→ Alertmanager ALSO posts a follow-up message when
  the alert resolves (turns green ✓), so the channel
  shows the full lifecycle, not just the failure
```

---

## Email Receiver

```yaml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: '<password>'

receivers:
  - name: 'team-email'
    email_configs:
      - to: 'platform-team@example.com'
        send_resolved: true
```

**Visual:**
```
Use case: low-urgency alerts, compliance/audit trails,
or teams without Slack/PagerDuty integration.

Not recommended as the ONLY channel for critical/paging
alerts - email lacks acknowledgment/escalation features
that PagerDuty/Opsgenie provide.
```

---

## PagerDuty Receiver

```yaml
receivers:
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - service_key: '<integration-key>'
        severity: '{{ .CommonLabels.severity }}'
        description: '{{ .CommonAnnotations.summary }}'
```

**Visual:**
```
Prometheus alert (severity: critical)
        │
        ▼
Alertmanager → PagerDuty
        │
        ▼
┌────────────────────────────────────┐
│ PagerDuty Incident Created             │
│  Escalation policy triggers               │
│  On-call engineer's phone rings              │
│  If unacknowledged in N min → escalates         │
│  to secondary on-call automatically                │
└────────────────────────────────────────┘

This is the standard receiver for anything that
needs a HUMAN woken up immediately - Slack/email
alone can't guarantee someone sees it at 3 AM.
```

---

## Opsgenie Receiver

```yaml
receivers:
  - name: 'opsgenie-oncall'
    opsgenie_configs:
      - api_key: '<api-key>'
        message: '{{ .CommonAnnotations.summary }}'
        priority: 'P1'
```

**Visual:**
```
Functionally similar to PagerDuty - alternative
incident management / on-call escalation tool.
Choose based on what your organization already uses
for on-call scheduling.
```

---

## Generic Webhook Receiver

**Sends the alert payload as JSON to any HTTP endpoint - the escape hatch for integrating with tools that don't have native Alertmanager support.**

```yaml
receivers:
  - name: 'custom-webhook'
    webhook_configs:
      - url: 'http://my-internal-service.svc.cluster.local:8080/alerts'
        send_resolved: true
```

**Visual:**
```
Alertmanager POSTs:
{
  "status": "firing",
  "alerts": [{
    "labels": {"alertname": "HighErrorRate", "severity": "critical"},
    "annotations": {"summary": "..."},
    "startsAt": "2026-07-14T10:00:00Z"
  }]
}
        │
        ▼
Your custom service can:
- Create a Jira ticket
- Post to Microsoft Teams (no native receiver exists)
- Trigger a custom auto-remediation script
- Forward to an internal ChatOps bot

Use case: ANY integration Alertmanager doesn't support
natively - webhook is the universal adapter.
```

---

## Custom Notification Templates

**Alertmanager uses Go templates to control exactly what a notification message looks like - customize beyond the defaults for clearer, more actionable alerts.**

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

```gotemplate
{{ define "slack.custom.text" }}
{{ range .Alerts }}
*Alert:* {{ .Labels.alertname }}
*Severity:* {{ .Labels.severity }}
*Namespace:* {{ .Labels.namespace }}
*Summary:* {{ .Annotations.summary }}
*Runbook:* {{ .Annotations.runbook_url }}
{{ end }}
{{ end }}
```

```yaml
receivers:
  - name: 'platform-team-slack'
    slack_configs:
      - channel: '#platform-alerts'
        text: '{{ template "slack.custom.text" . }}'
```

**Visual:**
```
Default message:                    Custom templated message:
┌───────────────────┐              ┌───────────────────────┐
│ HighErrorRate           │              │ 🔴 Alert: HighErrorRate     │
│ (raw labels dump)          │              │ Severity: critical             │
└───────────────────┘              │ Namespace: my-app                │
                                    │ Summary: Error rate > 5%           │
                                    │ Runbook: [link to wiki page]         │
                                    └───────────────────────┘

Always include a runbook_url annotation on your
Prometheus alerting rules and surface it in the
template - it turns "something's wrong" into
"here's exactly what to do about it."
```

---

## Multiple Receivers per Alert

**Combine `continue: true` routing (see 01routing_and_grouping.md) with distinct receivers to fan out a single alert to multiple destinations.**

**Visual:**
```
Critical Alert
      │
      ├──────────────→ pagerduty-oncall     (wakes someone up)
      │
      ├──────────────→ critical-slack-channel (team visibility)
      │
      └──────────────→ custom-webhook          (auto-creates a Jira ticket)

Each of these is a SEPARATE receiver, connected via
continue: true routes, all triggered from the SAME
underlying Prometheus alert.
```

---

## Visual Summary

```
Receiver Type    Best For
──────────────────────────────────────────────
slack_configs      Team visibility, non-urgent alerts
email_configs       Compliance/audit trail, low urgency
pagerduty_configs     Critical alerts needing a human woken up
opsgenie_configs       Alternative to PagerDuty
webhook_configs         Any custom integration (Jira, Teams, ChatOps)

Always pair receivers with:
- send_resolved: true  → so channels show the FULL lifecycle
- Custom templates        → so messages are actionable, not just raw labels
- runbook_url annotation    → so responders know what to actually DO
```

---

This guide covers Alertmanager's receiver types and notification customization - Slack, email, PagerDuty, webhooks, and templating - with visual representations of each integration.