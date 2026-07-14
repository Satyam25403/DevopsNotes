# Alertmanager Silencing & Inhibition - Visual Guide

Suppressing noise intelligently - temporary manual silences and automatic rules that stop alert storms from cascading into chaos.

## Table of Contents
- [Silences vs Inhibition](#silences-vs-inhibition)
- [Creating a Silence via UI](#creating-a-silence-via-ui)
- [Creating a Silence via amtool](#creating-a-silence-via-amtool)
- [Silence Matchers and Expiry](#silence-matchers-and-expiry)
- [Inhibition Rules](#inhibition-rules)
- [Common Inhibition Patterns](#common-inhibition-patterns)
- [Mute Timings (Time-Based Silencing)](#mute-timings-time-based-silencing)

---

## Silences vs Inhibition

**Visual:**
```
Silence                              Inhibition
┌──────────────────────┐            ┌──────────────────────┐
│ MANUALLY created         │            │ AUTOMATIC, config-driven │
│ Temporary (has an expiry)   │            │ Always active              │
│ "I know about this,           │            │ "This alert implies         │
│  stop notifying me"              │            │  that one doesn't matter"    │
│ Example: planned maintenance       │            │ Example: node down →          │
│  window                              │            │  suppress all pod-down          │
│                                       │            │  alerts ON that node               │
└──────────────────────┘            └──────────────────────┘

Both REDUCE notification noise, but for different reasons:
Silence     = a human decided to mute something temporarily
Inhibition   = the SYSTEM recognizes one alert makes another redundant
```

---

## Creating a Silence via UI

**Visual:**
```
Alertmanager UI → Alerts tab → find the alert → "Silence" button
┌────────────────────────────────────────┐
│ Create Silence                              │
│  Matchers: alertname="HighCPU"                 │
│            namespace="my-app"                     │
│  Starts: now                                          │
│  Ends: in 2 hours                                         │
│  Creator: priya                                              │
│  Comment: "Known issue, deploying fix, JIRA-1234"                │
└────────────────────────────────────────────┘

While active: matching alerts still FIRE in Prometheus
             and show in Alertmanager's UI, but NO
             notification is sent to any receiver.
```

---

## Creating a Silence via amtool

**Scriptable/CI-friendly way to create silences - useful for automating "silence alerts during deploy" patterns.**

```bash
amtool silence add \
  alertname="HighCPU" namespace="my-app" \
  --duration="2h" \
  --comment="Known issue, deploying fix, JIRA-1234" \
  --author="priya"
```

**Output Example:**
```
abc123-def456-ghi789    ← silence ID, save this to expire it early
```

### List and expire silences

```bash
amtool silence query
amtool silence expire abc123-def456-ghi789
```

**Visual:**
```
CI/CD Pipeline Pattern:
┌──────────────────────────────────────────┐
│ 1. Before deploy: amtool silence add \        │
│    namespace="my-app" --duration="15m"           │
│ 2. Deploy the new version                            │
│ 3. Run smoke tests                                       │
│ 4. amtool silence expire <id>  (end silence early,          │
│    or let it auto-expire after 15m)                             │
└──────────────────────────────────────────┘

Purpose: deploys often cause brief, EXPECTED alert
noise (pods restarting, momentary error spikes) -
silence it deliberately instead of getting paged
for a non-issue.
```

---

## Silence Matchers and Expiry

**Visual:**
```
Matcher precision matters:

Too broad:
alertname=".*"                    ← silences EVERYTHING, dangerous

Too narrow:
alertname="HighCPU" pod="pod-abc123"  ← only silences ONE specific pod,
                                          new pods after a restart
                                          won't be covered

Right balance:
alertname="HighCPU" namespace="my-app"   ← silences the alert type
                                            for the whole namespace,
                                            survives pod restarts

Expiry:
--duration="2h"           → relative duration
--start="2026-07-14T10:00:00Z" --end="2026-07-14T12:00:00Z"  → absolute window

⚠️ Silences do NOT auto-extend - if your maintenance
   runs long, you must extend or recreate the silence,
   or notifications resume automatically.
```

---

## Inhibition Rules

**Automatically suppresses a "target" alert when a related "source" alert is already firing - prevents cascading noise from a single root cause.**

```yaml
inhibit_rules:
  - source_match:
      alertname: 'NodeDown'
    target_match:
      alertname: 'PodDown'
    equal: ['node']
```

**Visual:**
```
Without inhibition:
Node worker-1 goes down
        │
        ▼
┌────────────────────────────────────┐
│ NodeDown{node=worker-1}                │
│ PodDown{node=worker-1, pod=app-1}         │
│ PodDown{node=worker-1, pod=app-2}            │
│ PodDown{node=worker-1, pod=app-3}               │
└────────────────────────────────────────┘
4 separate notifications - noisy, and the 3 PodDown
alerts don't add new information (of course pods are
down, the NODE is down)

With inhibition:
Node worker-1 goes down
        │
        ▼
┌────────────────────────────────────┐
│ NodeDown{node=worker-1}  ← ONLY this fires  │
│ (PodDown alerts for the same node are          │
│  automatically suppressed)                        │
└────────────────────────────────────────┘

equal: ['node']  ensures the suppression only applies
when BOTH alerts share the same node label value -
a PodDown on a healthy node still notifies normally.
```

---

## Common Inhibition Patterns

```yaml
inhibit_rules:
  # Suppress warning if critical of same alert is already firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']

  # Suppress all app alerts during a cluster-wide outage
  - source_match:
      alertname: 'ClusterUnreachable'
    target_match_re:
      alertname: '.*'
    equal: ['cluster']

  # Suppress dependent service alerts when the upstream is down
  - source_match:
      alertname: 'DatabaseDown'
    target_match:
      alertname: 'HighAPILatency'
    equal: ['namespace']
```

**Visual:**
```
Pattern 1: severity escalation
  critical firing  →  suppress the "duplicate" warning for
                       the SAME underlying issue

Pattern 2: blast-radius suppression
  cluster-wide outage → suppress EVERY other alert
                          (target_match_re: '.*') since
                          they're all just symptoms

Pattern 3: dependency-aware suppression
  DatabaseDown  →  suppress HighAPILatency (of course
                    latency is high if the DB is down -
                    that's not new information)
```

---

## Mute Timings (Time-Based Silencing)

**Newer Alertmanager feature (0.24+) - declaratively silence alerts during recurring time windows (maintenance windows, weekends) without manually creating a silence each time.**

```yaml
mute_time_intervals:
  - name: weekend-maintenance
    time_intervals:
      - weekdays: ['saturday', 'sunday']

route:
  receiver: 'default-slack'
  mute_time_intervals:
    - weekend-maintenance
```

**Visual:**
```
Saturday/Sunday                     Monday-Friday
┌──────────────────────┐           ┌──────────────────────┐
│ Alerts still fire in       │           │ Normal notification         │
│ Prometheus/Alertmanager       │           │ behavior                        │
│ UI, but NO notification is       │           └──────────────────────┘
│ SENT (matches mute window)          │
└──────────────────────┘

Use case: recurring low-priority maintenance windows,
or muting non-critical alerts outside business hours
WITHOUT needing to manually create/expire a silence
every single week.
```

---

## Visual Summary

```
1. Silence           → manual, temporary, has an expiry, human-initiated
2. amtool silence add  → scriptable silences for CI/CD deploy windows
3. Inhibition rule       → automatic, config-driven, always active
4. equal: [...]            → scopes suppression to matching label values only
5. Common patterns             → severity escalation, blast-radius, dependency-aware
6. mute_time_intervals            → recurring scheduled silencing (weekends, off-hours)
```

---

This guide covers Alertmanager's noise-reduction mechanisms - manual silences and automatic inhibition rules - that keep alert storms from overwhelming on-call responders, with visual representations of each pattern.