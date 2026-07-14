# Alertmanager Routing & Grouping - Visual Guide

How Alertmanager decides WHICH alerts go WHERE, and how it batches related alerts into single notifications.

## Table of Contents
- [The Route Tree](#the-route-tree)
- [Matchers](#matchers)
- [Nested Routes and continue](#nested-routes-and-continue)
- [group_by - Batching Related Alerts](#group_by---batching-related-alerts)
- [group_wait, group_interval, repeat_interval](#group_wait-group_interval-repeat_interval)
- [Real-World Routing Example](#real-world-routing-example)
- [Testing Routes with amtool](#testing-routes-with-amtool)

---

## The Route Tree

**Every alert enters at the top-level route and flows down through nested routes until it finds a match - like a decision tree.**

```yaml
route:
  receiver: 'default-slack'         # fallback if nothing else matches
  group_by: ['alertname']
  routes:
    - match:
        team: payments
      receiver: 'payments-slack'
    - match:
        team: platform
      receiver: 'platform-slack'
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
```

**Visual:**
```
                    Alert arrives
                          │
                          ▼
              ┌───────────────────────┐
              │   Top-level route        │
              │   (default-slack)          │
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  team: payments?    team: platform?   severity: critical?
        │                 │                 │
        ▼                 ▼                 ▼
  payments-slack     platform-slack    pagerduty-oncall

If NO child route matches → falls through to the
top-level receiver (default-slack) as a safety net
```

---

## Matchers

**Two syntaxes exist - the newer `matchers` list (recommended) and the older `match`/`match_re` maps.**

```yaml
# Newer syntax (Alertmanager 0.22+, recommended)
routes:
  - matchers:
      - severity="critical"
      - team=~"payments|billing"
    receiver: 'critical-payments'

# Older syntax (still widely seen in existing configs)
routes:
  - match:
      severity: critical
    match_re:
      team: payments|billing
    receiver: 'critical-payments'
```

**Visual:**
```
matchers: - severity="critical"     → exact match
          - team=~"payments|billing" → regex match (=~)
          - severity!="warning"       → negative match (!=)
          - team!~"test.*"             → negative regex (!~)

Prefer the matchers: list syntax in new configs - it's
more expressive and is the direction Alertmanager is
standardizing on.
```

---

## Nested Routes and continue

**By default, an alert stops at the FIRST matching route. Use `continue: true` to let it also match subsequent sibling/child routes.**

```yaml
route:
  receiver: 'default-slack'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
      continue: true        # keep evaluating further routes
    - match:
        severity: critical
      receiver: 'critical-slack-channel'
```

**Visual:**
```
Without continue: true
Alert (severity: critical)
        │
        ▼
  Matches route 1 → pagerduty-oncall
        │
        ✗ STOPS HERE, route 2 never evaluated

With continue: true
Alert (severity: critical)
        │
        ▼
  Matches route 1 → pagerduty-oncall  (sent)
        │
        ▼ (continues)
  Matches route 2 → critical-slack-channel  (ALSO sent)

Use continue: true when you want the SAME alert to
notify multiple destinations (e.g. page on-call AND
post to a visibility channel).
```

---

## group_by - Batching Related Alerts

**Combines multiple alerts that share the same label values into a SINGLE notification, instead of spamming one message per alert.**

```yaml
route:
  group_by: ['alertname', 'namespace']
```

**Visual:**
```
Without grouping:
Prometheus fires:
  HighCPU{pod=pod-1}
  HighCPU{pod=pod-2}
  HighCPU{pod=pod-3}
        │
        ▼
3 separate Slack messages

With group_by: ['alertname']
Prometheus fires the same 3 alerts
        │
        ▼
1 Slack message:
"HighCPU firing for 3 instances: pod-1, pod-2, pod-3"

group_by: ['...']  (special value)
→ groups by ALL labels (effectively no grouping,
  every unique label combination gets its own group)

group_by: []
→ groups EVERYTHING into a single notification,
  regardless of alertname (rarely what you want)
```

---

## group_wait, group_interval, repeat_interval

**Three timers that control notification pacing - getting these wrong is the #1 cause of either alert spam or missed alerts.**

```yaml
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**Visual:**
```
group_wait: 30s
┌──────────────────────────────────────┐
│ First alert in a NEW group arrives       │
│ Wait 30s to see if related alerts join     │
│ THEN send the first notification              │
└──────────────────────────────────────────┘
Purpose: batch a burst of related alerts into
         one notification instead of firing instantly

group_interval: 5m
┌──────────────────────────────────────────┐
│ Group already notified once                    │
│ New alert joins the SAME group                    │
│ Wait at least 5m before sending an UPDATED           │
│ notification with the new alert included                │
└──────────────────────────────────────────────┘
Purpose: don't re-notify on every single new
         alert joining an already-active group

repeat_interval: 4h
┌──────────────────────────────────────────┐
│ Alert is STILL firing, nothing changed         │
│ Re-send the notification every 4h as a             │
│ reminder that it's still broken                        │
└──────────────────────────────────────────────┘
Purpose: don't let an unresolved critical alert
         go silent for days just because nothing
         "changed" - but don't spam every minute either
```

**Typical production values:**
```
Critical/paging route:    group_wait: 10s,  repeat_interval: 1h
Warning/Slack-only route: group_wait: 1m,   repeat_interval: 12h
Low-priority/ticket route: group_wait: 5m,  repeat_interval: 24h
```

---

## Real-World Routing Example

```yaml
route:
  receiver: 'default-slack'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers:
        - severity="critical"
      receiver: 'pagerduty-oncall'
      group_wait: 10s
      repeat_interval: 1h
      continue: true
    - matchers:
        - severity="critical"
      receiver: 'critical-slack-channel'
    - matchers:
        - team="payments"
      receiver: 'payments-team-slack'
    - matchers:
        - team="platform"
      receiver: 'platform-team-slack'
    - matchers:
        - alertname="Watchdog"
      receiver: 'null'          # silently drop health-check heartbeat alerts
```

**Visual:**
```
                          Alert arrives
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
      severity=critical?   team=payments?    alertname=Watchdog?
             │                  │                  │
    ┌────────┴────────┐        ▼                  ▼
    ▼                 ▼   payments-team-      receiver: 'null'
pagerduty-oncall  critical-    slack          (dropped entirely -
(paged, 1h repeat) slack-channel                used for the
    │              (also sent,                   "everything's fine"
    ▼               visibility)                   heartbeat alert)
 continues to
 next route too
```

---

## Testing Routes with amtool

**Preview which receiver an alert with given labels would be routed to - WITHOUT actually sending a notification.**

```bash
amtool config routes test \
  --config.file=alertmanager.yml \
  severity=critical team=payments
```

**Output Example:**
```
pagerduty-oncall
critical-slack-channel
```

**Visual:**
```
Always test a routing change BEFORE deploying it:
1. amtool config routes test --config.file=alertmanager.yml <labels>
2. Confirm the expected receiver(s) show up
3. THEN apply/reload the config

This catches "oops, I broke the route tree and now
nothing pages on-call" before it becomes a real incident.
```

---

## Visual Summary

```
1. route            → the decision tree, top-down evaluation
2. matchers            → exact/regex conditions on alert labels
3. continue: true         → allow an alert to match MULTIPLE routes
4. group_by                → batch related alerts into one notification
5. group_wait                → initial batching delay for new groups
6. group_interval               → delay before re-notifying an updated group
7. repeat_interval                 → how often to re-remind about a still-firing alert
8. amtool config routes test         → dry-run test any routing change before deploying
```

---

This guide covers Alertmanager's routing tree and grouping mechanics - the core logic that decides who gets notified, when, and how often, with visual representations of each concept.