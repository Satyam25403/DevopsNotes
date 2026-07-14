# Chaos Mesh Workflows & Scheduling - Orchestrating Multi-Step Chaos

How to chain multiple experiments together and run chaos continuously/automatically, with visual diagrams.

## Table of Contents
- [Why Workflows](#why-workflows)
- [Workflow CRD - Serial Steps](#workflow-crd---serial-steps)
- [Workflow CRD - Parallel Steps](#workflow-crd---parallel-steps)
- [Conditional / Suspend Steps](#conditional--suspend-steps)
- [Schedule CRD - Recurring Chaos](#schedule-crd---recurring-chaos)
- [GameDay Pattern](#gameday-pattern)
- [Combining with Observability](#combining-with-observability)

---

## Why Workflows

**A single experiment answers one question. Real incidents are rarely a single fault - a Workflow lets you compose realistic, multi-step failure scenarios.**

**Visual:**
```
Single Experiment:
"What happens if backend loses network for 1 minute?"

Workflow (realistic incident simulation):
"What happens if backend loses network for 1 minute,
 THEN CPU spikes to 90%,
 THEN a pod gets killed,
 all while load is normal?"

This mirrors how real production outages actually cascade -
rarely one clean failure, usually a chain reaction.
```

---

## Workflow CRD - Serial Steps

**Runs experiments one after another, in order.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  name: cascading-failure-test
  namespace: my-app
spec:
  entry: the-entry
  templates:
    - name: the-entry
      templateType: Serial
      deadline: 10m
      children:
        - network-delay-step
        - cpu-stress-step
        - pod-kill-step

    - name: network-delay-step
      templateType: NetworkChaos
      deadline: 2m
      networkChaos:
        action: delay
        mode: all
        selector:
          namespaces: ["my-app"]
          labelSelectors: { app: backend }
        delay:
          latency: "200ms"

    - name: cpu-stress-step
      templateType: StressChaos
      deadline: 2m
      stressChaos:
        mode: one
        selector:
          namespaces: ["my-app"]
          labelSelectors: { app: backend }
        stressors:
          cpu: { workers: 2, load: 90 }

    - name: pod-kill-step
      templateType: PodChaos
      deadline: 30s
      podChaos:
        action: pod-kill
        mode: one
        selector:
          namespaces: ["my-app"]
          labelSelectors: { app: backend }
```

**Visual:**
```
Timeline:
0min ─────────────2min─────────────4min─────────4.5min
  │ network-delay │  cpu-stress    │ pod-kill  │
  │ (200ms added) │  (90% CPU)     │ (kill 1)  │
  └───────────────┴────────────────┴───────────┘
       Step 1            Step 2         Step 3
   (runs, finishes,   (runs, finishes, (runs, finishes)
    then next starts)  then next starts)

Serial = one step must fully complete before the next begins
```

---

## Workflow CRD - Parallel Steps

**Runs multiple experiments at the same time - simulates compound, simultaneous failures.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  name: compound-failure-test
  namespace: my-app
spec:
  entry: the-entry
  templates:
    - name: the-entry
      templateType: Parallel
      deadline: 5m
      children:
        - network-delay-step
        - cpu-stress-step
```

**Visual:**
```
Timeline:
0min ──────────────────────────────5min
  │ network-delay-step (running)   │
  │ cpu-stress-step    (running)   │
  └─────────────────────────────────┘
        Both run AT THE SAME TIME

Parallel = simulates realistic "everything breaks at once"
scenarios (e.g. a node failure causing BOTH network and
resource pressure simultaneously)
```

---

## Conditional / Suspend Steps

**Insert pauses between steps to observe recovery time, or gate progression on manual approval.**

```yaml
templates:
  - name: the-entry
    templateType: Serial
    children:
      - pod-kill-step
      - wait-and-observe
      - verify-step

  - name: wait-and-observe
    templateType: Suspend
    deadline: 2m
```

**Visual:**
```
pod-kill-step  →  Suspend (2min pause)  →  verify-step
   fault            observe recovery         check results
   injected          time / alerting          (manual or
                                                automated)

Use case: giving your monitoring/alerting pipeline time to
fire, and giving on-call a window to confirm the runbook works
```

---

## Schedule CRD - Recurring Chaos

**Runs an experiment on a cron-like schedule automatically - moves chaos engineering from "one-off game day" to continuous practice.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: nightly-pod-kill
  namespace: my-app
spec:
  schedule: "0 2 * * *"          # every night at 2 AM
  type: "PodChaos"
  historyLimit: 5
  concurrencyPolicy: Forbid       # don't overlap runs
  podChaos:
    action: pod-kill
    mode: fixed-percent
    value: "20"
    selector:
      namespaces: ["my-app"]
      labelSelectors:
        app: backend
```

**Visual:**
```
Every night 02:00
┌─────────────────────────────────────────┐
│ Schedule: nightly-pod-kill                │
│ → creates a new PodChaos experiment       │
│ → kills 20% of backend pods               │
│ → records result in dashboard history     │
└─────────────────────────────────────────┘

concurrencyPolicy:
Forbid    → skip this run if the last one is still active
Allow     → run overlapping instances anyway
Forbid is the safer default for production
```

**Check schedule history:**
```bash
kubectl get schedule nightly-pod-kill -n my-app
kubectl get podchaos -n my-app --selector=managed-by=nightly-pod-kill
```

---

## GameDay Pattern

**A structured, human-run exercise where a team deliberately triggers chaos and practices incident response - the classic Netflix-style chaos engineering ritual.**

**Visual:**
```
GameDay Runbook:
┌──────────────────────────────────────────────────┐
│ 1. Announce window to stakeholders                  │
│ 2. Confirm steady-state dashboards are up            │
│    (success rate, latency, error budget)             │
│ 3. Inject chaos (Workflow: cascading-failure-test)   │
│ 4. On-call responds AS IF it were real                │
│ 5. Time-to-detect, time-to-mitigate measured          │
│ 6. Debrief: what broke, what worked, what to fix       │
│ 7. File tickets for weaknesses found                  │
└──────────────────────────────────────────────────┘

Cadence: monthly or quarterly for critical services
Scope: start in staging, graduate to production once
       confident (with tight blast-radius controls - see
       04production_safety_and_rbac.md)
```

---

## Combining with Observability

**Chaos experiments are only useful if you can SEE the impact - always pair with your metrics stack.**

**Visual:**
```
Chaos Mesh Event                    Your Observability Stack
┌───────────────────┐              ┌───────────────────────┐
│ PodChaos: pod-kill  │  ────────→  │ Prometheus: error rate │
│ started at 14:00:00 │              │  spike detected 14:00:03│
└───────────────────┘              │ Grafana: latency graph  │
                                    │  shows p99 spike        │
                                    │ Alertmanager: paged      │
                                    │  on-call at 14:00:15     │
                                    └───────────────────────┘

Overlay chaos experiment start/end timestamps as annotations
on Grafana dashboards to correlate cause and effect precisely.
```

---

## Visual Summary

```
1. Single experiment       → answers ONE resilience question
2. Workflow (Serial)       → chains steps, one after another
3. Workflow (Parallel)     → simulates simultaneous failures
4. Suspend step            → pause to observe recovery
5. Schedule                → recurring, automated chaos (nightly/weekly)
6. GameDay                 → human-run exercise, practices incident response
7. Always pair with        → Prometheus/Grafana/Alertmanager to measure impact
```

---

This guide covers orchestrating Chaos Mesh experiments into realistic multi-step scenarios using Workflows and Schedules, with visual representations of each pattern.