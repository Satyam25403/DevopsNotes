# Grafana - Alerting

Turning queries into proactive notifications: alert rules, contact points, notification policies, and silences.

## Table of Contents
- [Why Alerting Matters](#why-alerting-matters)
- [Grafana Alerting Architecture](#grafana-alerting-architecture)
- [Creating an Alert Rule](#creating-an-alert-rule)
- [Evaluation and For Duration](#evaluation-and-for-duration)
- [Contact Points](#contact-points)
- [Notification Policies](#notification-policies)
- [Silences](#silences)
- [Multi-Dimensional Alerts](#multi-dimensional-alerts)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Why Alerting Matters

**Visual:**
```
Without Alerting:
Dashboard shows a problem → but only if someone happens
to be looking at it right now → outage continues silently
overnight until customers complain → team finds out at 9 AM,
hours after it started 🔥

With Alerting:
Threshold breached → notification fires within minutes →
on-call engineer paged → issue addressed before most
customers even notice
```

**The core principle:** a dashboard is passive (someone has to look at it); an alert is active (it comes to you).

---

## Grafana Alerting Architecture

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│                    Alert Rule                            │
│   "Fire if error_rate > 5% for 5 minutes straight"          │
└───────────────────────┬───────────────────────────────┘
                         │ evaluates on a schedule
                         ↓
                  Condition breached?
                         │
                 ┌───────┴────────┐
                 ↓                  ↓
              No                  Yes
        (stay Normal)        (state → Pending → Firing)
                                    │
                                    ↓
                         ┌─────────────────────┐
                         │ Notification Policy     │
                         │ (routes based on labels)  │
                         └──────────┬──────────────┘
                                    ↓
                         ┌─────────────────────┐
                         │  Contact Point           │
                         │  (Slack, PagerDuty,       │
                         │   Email, Webhook, etc.)     │
                         └─────────────────────┘
```

---

## Creating an Alert Rule

```
Alerting → Alert rules → New alert rule
```

**Configuration:**
```
Rule name: High Error Rate - Payments Service

Query (A):
sum(rate(http_requests_total{service="payments", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="payments"}[5m]))
* 100

Condition: WHEN Query A IS ABOVE 5

Evaluation:
  Evaluate every: 1m
  For: 5m
```

**Visual:**
```
What this rule does:
1. Every 1 minute, run the error-rate percentage query
2. If the result is above 5, mark as "Pending"
3. If it STAYS above 5 continuously for 5 minutes, mark as "Firing"
4. Only THEN does it actually notify anyone

Why not fire instantly on the first breach?
A single 10-second blip above 5% (e.g. one bad request
during a deploy) shouldn't wake someone up at 3 AM —
the "For" duration filters out noise/flapping.
```

---

## Evaluation and For Duration

**Visual:**
```
Error Rate Over Time:
8% │      ╱╲
6% │     ╱  ╲        ╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
5% │────╱────╲──────╱─────────────────── ← threshold
4% │   ╱      ╲    ╱
2% │  ╱        ╲__╱
   └──────────────────────────────────→ time
       ↑                ↑
   brief spike      sustained breach
   (Pending → back    (Pending → Firing
    to Normal,          after 5m, NOTIFIES)
    no notification)

The brief spike never notifies because it didn't stay
above threshold for the full "For: 5m" duration.
The sustained breach DOES notify, because it held.
```

---

## Contact Points

**A Contact Point defines WHERE a notification actually gets sent.**

```
Alerting → Contact points → New contact point

Name: platform-team-slack
Integration: Slack
Webhook URL: https://hooks.slack.com/services/...
```

**Visual:**
```
┌───────────────────────────────────────────────┐
│ Contact Point Type      Typical Use                 │
├───────────────────────────────────────────────┤
│ Slack                     Team channel notifications      │
│ PagerDuty/Opsgenie          On-call paging, escalation        │
│ Email                       Less urgent/summary notifications   │
│ Webhook                      Custom integrations, ticketing        │
│                            systems, ChatOps bots                     │
└───────────────────────────────────────────────┘
```

**Visual — message template customization:**
```
Default alert message is often too generic.
Custom templates can include:
- The exact metric value that breached
- A direct link back to the dashboard
- Labels identifying which specific service/pod/host

{{ .CommonLabels.service }} error rate is
{{ .Values.A }}% (threshold: 5%)
→ View dashboard: https://grafana.company.com/d/abc123
```

---

## Notification Policies

**Notification Policies route firing alerts to the correct Contact Point, based on labels — this is how different alerts reach different teams.**

**Visual:**
```
┌─────────────────────────────────────────────────┐
│               Notification Policy Tree                │
├─────────────────────────────────────────────────┤
│  Default Policy → contact: general-slack-channel     │
│                                                    │
│  ├── Match: team=payments                              │
│  │    → contact: payments-team-pagerduty                  │
│  │                                                     │
│  ├── Match: team=platform                                │
│  │    → contact: platform-team-slack                        │
│  │                                                     │
│  └── Match: severity=critical                              │
│       → contact: exec-escalation-pagerduty (ALSO fires,       │
│         in addition to the team-specific route above)           │
└─────────────────────────────────────────────────┘
```

**Visual — how labels drive routing:**
```
Alert Rule has labels: team=payments, severity=critical
                              ↓
         Notification policy tree evaluates label matches
                              ↓
   Matches "team=payments" AND "severity=critical"
                              ↓
      Notification sent to BOTH payments-team-pagerduty
      AND exec-escalation-pagerduty (nested policies can
      route to multiple contact points for the same alert)
```

---

## Silences

**A Silence temporarily suppresses notifications for alerts matching specific labels — without disabling the underlying alert rule.**

**Visual:**
```
Scenario: Planned maintenance window on the payments database
for 2 hours, during which connection errors are EXPECTED.

Without a Silence:
Maintenance starts → alert rule fires as designed →
   on-call engineer gets paged for an "issue" that's
   actually just expected, planned maintenance 😩

With a Silence:
Silence created: matcher "service=payments-db", duration 2h
   → alerts matching this label combination are suppressed
   → maintenance proceeds without unnecessary pages
   → silence automatically expires after 2 hours,
     alerting resumes normal operation automatically
```

```
Alerting → Silences → New silence
Matching labels: service=payments-db
Duration: 2h
Comment: "Planned DB maintenance - JIRA-1234"
```

---

## Multi-Dimensional Alerts

**One alert rule can fire separately for EACH matching series (e.g., one alert per pod, per host, per service) instead of just one blob alert.**

**Visual:**
```
Query: rate(http_requests_total{status=~"5.."}[5m]) by (service)

Instead of ONE combined result, this returns MULTIPLE
series — one per service label value:
  {service="payments"} = 8%
  {service="auth"}      = 2%
  {service="search"}     = 12%

Grafana evaluates the alert condition PER series:
  payments: 8% > 5% → FIRING (separate alert instance)
  auth:      2% > 5% → Normal
  search:     12% > 5% → FIRING (separate alert instance)

Result: TWO distinct firing alerts, each carrying its
own "service" label — so notifications and routing can
be specific to which service is actually affected.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up alerting for a platform with 20 microservices, each owned by a different team, and wants alerts to reach the RIGHT team without creating alert fatigue.

**What they configure:**
1. A **single multi-dimensional alert rule** for error rate, using `by (service, team)` in the query, so one rule definition covers all 20 services rather than maintaining 20 near-duplicate rules.
2. A **notification policy tree** matching on the `team` label, routing each firing alert to that specific team's Slack channel — the payments team never gets paged for the search team's issues, and vice versa.
3. A **separate, more aggressive policy branch** matching `severity=critical` that ALSO escalates to a PagerDuty rotation, regardless of which team owns the service — ensuring genuinely severe issues reach an on-call human even outside business hours, while routine warnings just post to Slack.
4. A **"For: 5m" duration** on all warning-level alerts (to avoid noise from transient blips) but a **shorter "For: 1m"** on critical-level alerts like "service is completely down" — where even a brief outage deserves fast notification.
5. Trains teams to create **Silences** (with a mandatory comment linking to a ticket) during planned maintenance windows, rather than the old habit of just disabling the entire alert rule and sometimes forgetting to re-enable it afterward.

**Why this matters:** Alerting that pages the wrong team, or pages everyone for everything, quickly leads to alert fatigue — where engineers start ignoring or muting notifications entirely, defeating the whole purpose. Precise label-based routing is what keeps alerting trustworthy and actionable at scale.

---

Next: **05advanced_realworld_usecases.md** — dashboards-as-code (provisioning), plugins, and mature organization-wide Grafana practices.