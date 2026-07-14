# Chaos Mesh Dashboard & Observability - Visual Guide

Using the built-in Chaos Dashboard and integrating experiment visibility into your monitoring stack.

## Table of Contents
- [Dashboard Overview](#dashboard-overview)
- [Creating Experiments via the Dashboard](#creating-experiments-via-the-dashboard)
- [Events and Archives](#events-and-archives)
- [chaosctl for Live Debugging](#chaosctl-for-live-debugging)
- [Prometheus Metrics from Chaos Mesh](#prometheus-metrics-from-chaos-mesh)
- [Grafana Annotations for Experiment Correlation](#grafana-annotations-for-experiment-correlation)
- [RBAC for the Dashboard](#rbac-for-the-dashboard)

---

## Dashboard Overview

```bash
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

**Visual:**
```
http://localhost:2333
┌───────────────────────────────────────────────────┐
│  Chaos Mesh Dashboard                              │
├───────────────────────────────────────────────────┤
│ ▸ Experiments   - active/paused fault injections    │
│ ▸ Workflows     - multi-step orchestrations         │
│ ▸ Schedules     - recurring chaos jobs              │
│ ▸ Events        - timeline of everything injected   │
│ ▸ Archives      - historical experiment results     │
│ ▸ Settings      - RBAC token management             │
└───────────────────────────────────────────────────┘
```

---

## Creating Experiments via the Dashboard

**A visual, form-based way to build experiments without hand-writing YAML - useful for onboarding teammates unfamiliar with the CRDs.**

**Visual:**
```
Dashboard Flow:
┌────────────────────────────────────┐
│ 1. Choose experiment type           │
│    (Pod / Network / Stress / ...)   │
│ 2. Choose target                    │
│    Namespace: my-app                │
│    Label selector: app=backend      │
│    Mode: fixed-percent 30%          │
│ 3. Configure the specific fault     │
│    (e.g. NetworkChaos → delay 200ms)│
│ 4. Set duration / schedule          │
│ 5. Preview generated YAML           │
│ 6. Submit                           │
└────────────────────────────────────┘

Dashboard always shows the underlying YAML before submission -
good practice: copy it into Git for version control even if
you built it via the UI.
```

---

## Events and Archives

### View events via CLI

```bash
kubectl get events -n my-app --field-selector involvedObject.kind=PodChaos
```

### View via dashboard

**Visual:**
```
Events Tab:
┌──────────────────────────────────────────────────────┐
│ TIME       TYPE          NAMESPACE   MESSAGE          │
│ 14:00:00   PodChaos       my-app     Started pod-kill │
│ 14:00:02   PodChaos       my-app     backend-1 killed │
│ 14:01:00   PodChaos       my-app     Experiment ended │
└──────────────────────────────────────────────────────┘

Archives Tab:
┌──────────────────────────────────────────────────────┐
│ Completed experiments with full YAML + timeline        │
│ Useful for post-incident review / audit trail          │
│ Export for compliance/change-management records         │
└──────────────────────────────────────────────────────┘
```

---

## chaosctl for Live Debugging

**Inspect what's actually happening at the daemon/pod level while an experiment runs.**

```bash
chaosctl logs
chaosctl debug <podchaos-name> -n my-app
```

**Output Example:**
```
Pod: backend-1
Chaos Type: NetworkChaos
Status: Injected
Debug info:
  tc qdisc show: netem delay 200ms 50ms
  iptables rules: DROP applied for partition test
```

**Visual:**
```
chaosctl talks directly to chaos-daemon on the node
to show you the RAW underlying commands (tc/iptables/eBPF)
that were actually applied - useful when an experiment
"succeeds" in the CRD status but you don't trust it actually
did anything.
```

---

## Prometheus Metrics from Chaos Mesh

**The controller-manager exposes its own Prometheus metrics about experiment execution.**

```yaml
# ServiceMonitor example (if using Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: chaos-mesh
  namespace: chaos-mesh
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller-manager
  endpoints:
    - port: http-metrics
```

**Visual:**
```
Key metrics exposed:
chaos_controller_manager_experiments_total
chaos_controller_manager_experiment_duration_seconds

Use these to build a "chaos activity" Grafana panel showing
how much/how often chaos has actually been run - useful for
proving chaos engineering maturity to stakeholders.
```

---

## Grafana Annotations for Experiment Correlation

**Overlay chaos experiment start/end times directly on your existing dashboards to visually correlate cause and effect.**

**Visual:**
```
Grafana Panel: Backend Success Rate
100% ┤
 99% ┤        ┌──┐
 98% ┤────────┤▼▼│──────────────
 97% ┤        └──┘
     └────────────────────────────→ time
              ↑
      Annotation: "PodChaos: kill-backend-pod started"
      ↑
      Success rate dip correlates exactly with chaos injection
      → confirms causality, not coincidence
```

**How:** send a webhook from a CI/CD or Workflow post-start hook to Grafana's annotations API when an experiment begins/ends.

---

## RBAC for the Dashboard

**Chaos Mesh dashboard access is token-based, scoped per namespace - critical for production safety.**

```bash
chaosctl dashboard rbac-token -n my-app --role=viewer
```

**Visual:**
```
Roles:
manager  → can create/edit/delete experiments in the namespace
viewer   → read-only, can see experiments/events but not run them

Team Setup Example:
┌────────────────────────────────────────┐
│ Namespace: staging   → role: manager     │  (devs can chaos freely)
│ Namespace: production → role: viewer      │  (devs can observe only)
│                                          │
│ Production chaos requires a separate,    │
│ audited process (see 04production_       │
│ safety_and_rbac.md)                       │
└────────────────────────────────────────┘
```

---

## Visual Summary

```
1. Dashboard         → visual experiment builder + history
2. Events            → real-time timeline of what was injected
3. Archives          → historical audit trail
4. chaosctl          → raw debugging (tc/iptables/eBPF level)
5. Prometheus        → metrics on chaos activity itself
6. Grafana annotations → correlate chaos timing with impact graphs
7. RBAC tokens        → scope who can run chaos in which namespace
```

---

This guide covers observing and auditing Chaos Mesh experiments through the dashboard, CLI debugging tools, and integration with Prometheus/Grafana, with visual representations of each workflow.