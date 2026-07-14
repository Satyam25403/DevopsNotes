# LitmusChaos Production Safety - RBAC, Blast Radius, Safeguards

How to run Litmus experiments in production without causing a real outage, with visual diagrams.

## Table of Contents
- [Why Safety Matters More Than the Fault Itself](#why-safety-matters-more-than-the-fault-itself)
- [chaosServiceAccount - Scoped Permissions](#chaosserviceaccount---scoped-permissions)
- [Blast Radius Control](#blast-radius-control)
- [Namespace Install Mode vs Cluster-Wide](#namespace-install-mode-vs-cluster-wide)
- [Duration Limits and FORCE Flag](#duration-limits-and-force-flag)
- [Abort / Kill Switch](#abort--kill-switch)
- [Projects, Environments, and RBAC in ChaosCenter](#projects-environments-and-rbac-in-chaoscenter)
- [Progressive Rollout of Chaos Practice](#progressive-rollout-of-chaos-practice)
- [Pre-Flight Checklist](#pre-flight-checklist)

---

## Why Safety Matters More Than the Fault Itself

**The goal is finding weaknesses safely - not recreating a real incident. Every experiment needs a scoped blast radius and a guaranteed way to stop.**

**Visual:**
```
Good Litmus Experiment                Bad Litmus Experiment
┌──────────────────────┐             ┌──────────────────────┐
│ Scoped chaosServiceAcct │             │ Cluster-admin service  │
│ Namespace-install agent  │             │  account used for chaos │
│ TOTAL_CHAOS_DURATION set │             │ Cluster-wide agent        │
│ Probes gating pass/fail   │             │ No probes - "ran" = pass  │
│ engineState: stop ready    │             │ No one knows how to abort │
└──────────────────────┘             └──────────────────────┘
     Controlled experiment                   Actual outage risk
```

---

## chaosServiceAccount - Scoped Permissions

**Every ChaosEngine runs under a specific ServiceAccount - this is your primary RBAC guardrail, controlling exactly what the experiment pod is allowed to touch.**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-delete-sa-role
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete", "list", "get"]
  - apiGroups: ["litmuschaos.io"]
    resources: ["chaosengines", "chaosexperiments", "chaosresults"]
    verbs: ["get", "list", "update", "patch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-delete-sa
  namespace: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-delete-sa-binding
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: pod-delete-sa
    namespace: my-app
roleRef:
  kind: Role
  name: pod-delete-sa-role
  apiGroup: rbac.authorization.k8s.io
```

```yaml
spec:
  chaosServiceAccount: pod-delete-sa    # scoped, not cluster-admin
```

**Visual:**
```
Bad:  chaosServiceAccount: cluster-admin
      → experiment pod CAN do anything in the cluster
      → a buggy/malicious experiment = full cluster compromise

Good: chaosServiceAccount: pod-delete-sa
      → experiment pod can ONLY delete pods in my-app
      → blast radius enforced at the Kubernetes RBAC layer,
        not just by convention
```

**ChaosHub experiments ship a default RBAC manifest** - always review it before applying, and tighten `resourceNames`/namespaces further for production use.

---

## Blast Radius Control

**Use `applabel` and specific tags to target the smallest useful slice, and prefer percentage-based tunables where the experiment supports them.**

```yaml
spec:
  appinfo:
    appns: my-app
    applabel: "app=backend,tier=canary"   # narrow selector
    appkind: deployment
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: PODS_AFFECTED_PERC
              value: "10"     # only 10% of matching pods
```

**Visual:**
```
Progression:
Week 1:  PODS_AFFECTED_PERC: "0"  (single pod default behavior)
Week 2:  PODS_AFFECTED_PERC: "10"
Week 3:  PODS_AFFECTED_PERC: "30"
Month 2: full deployment, staging only

Rule: same as any chaos tool - increase blast radius
      gradually, one variable at a time.
```

---

## Namespace Install Mode vs Cluster-Wide

**Chosen when onboarding an Agent - determines the maximum possible blast radius from a compromised or misconfigured agent.**

**Visual:**
```
Namespace-scoped Agent               Cluster-wide Agent
┌─────────────────────┐             ┌─────────────────────┐
│ Can only run chaos     │             │ Can run chaos in ANY   │
│ inside the namespace     │             │ namespace in the         │
│ it was installed into     │             │ cluster                   │
└─────────────────────┘             └─────────────────────┘

Recommendation:
Production   → namespace-scoped agent per critical namespace
Staging/Dev   → cluster-wide agent is acceptable
```

---

## Duration Limits and FORCE Flag

**Both prevent an experiment from running indefinitely or being unnecessarily destructive.**

```yaml
env:
  - name: TOTAL_CHAOS_DURATION
    value: "60"
  - name: FORCE
    value: "false"     # graceful termination, not SIGKILL
```

**Visual:**
```
TOTAL_CHAOS_DURATION: "60"
0s ───────────────────────60s
│  fault active            │ auto-stops, operator cleans up
└───────────────────────────┘

FORCE: "false"  → sends SIGTERM, respects preStop hooks/graceful shutdown
FORCE: "true"   → sends SIGKILL immediately (more realistic "hard crash"
                   test, but skips graceful shutdown paths - use deliberately,
                   not as a default)
```

---

## Abort / Kill Switch

**Stop a running experiment immediately by patching the ChaosEngine's state.**

```bash
kubectl patch chaosengine backend-pod-delete -n my-app \
  --type merge --patch '{"spec":{"engineState":"stop"}}'
```

**Or delete it outright:**

```bash
kubectl delete chaosengine backend-pod-delete -n my-app
```

**Visual:**
```
Emergency Abort Flow:
┌────────────────────────────────────────────┐
│ On-call sees unexpected impact                 │
│         │                                       │
│         ▼                                       │
│ kubectl patch chaosengine <name> -n <ns> \        │
│   --type merge --patch '{"spec":              │
│   {"engineState":"stop"}}'                      │
│         │                                       │
│         ▼                                       │
│ chaos-operator halts the running experiment pod   │
└────────────────────────────────────────────┘

Also available directly from the ChaosCenter UI:
"Stop Workflow" button on any in-progress run.

Always have this command (or the UI button location)
confirmed BEFORE starting any production experiment.
```

---

## Projects, Environments, and RBAC in ChaosCenter

**ChaosCenter has its own user/project layer on top of Kubernetes RBAC - controls who can even SEE or trigger chaos for a given team/environment.**

**Visual:**
```
ChaosCenter Structure:
┌────────────────────────────────────────────┐
│ Project: platform-team                        │
│  Members: Editor / Viewer roles per user         │
│  Environments:                                     │
│   - staging      (Non-Production type)              │
│   - production    (Production type - extra warnings) │
└────────────────────────────────────────────┘

Editor  → can create/run/stop workflows
Viewer  → read-only, can see resilience scores/results

Litmus explicitly flags an Environment as "Production" type
in the UI - this surfaces a confirmation warning before
running any workflow against it, an extra guardrail beyond
plain RBAC.
```

---

## Progressive Rollout of Chaos Practice

**Visual:**
```
Level 1: Local/Dev
  Run single ChaosHub experiments manually, learn probes

Level 2: Staging, Manual GameDay
  Team builds multi-step Workflows, runs on a schedule

Level 3: Staging, Automated + Scheduled
  Recurring cron-scheduled workflows, resilience score tracked

Level 4: Production, Scoped, Manual Approval
  Namespace-scoped agent, tight chaosServiceAccount,
  small PODS_AFFECTED_PERC, run with SRE present

Level 5: Production, Continuous
  Scheduled low-blast-radius workflows running continuously,
  gated by promProbe against real SLOs, auto-abort on breach
```

---

## Pre-Flight Checklist

```
☐ chaosServiceAccount is scoped (not cluster-admin)
☐ Agent install mode matches environment risk (namespace-scoped for prod)
☐ appinfo.applabel is as narrow as possible
☐ TOTAL_CHAOS_DURATION set (no indefinite runs)
☐ FORCE flag deliberately chosen (true/false), not left default by accident
☐ At least one probe configured (httpProbe/promProbe) - never run "blind"
☐ ChaosCenter Environment correctly marked Production (extra confirmation)
☐ Abort command or "Stop Workflow" button location confirmed beforehand
☐ Workflow YAML version-controlled in Git (GitOps applied)
☐ Dashboards (ChaosCenter resilience score + Grafana/Prometheus) watched live
```

---

## Visual Summary

```
Safety Layers (defense in depth):
1. ChaosCenter Project/Environment RBAC → who can see/trigger chaos at all
2. Agent install mode (namespace vs cluster-wide) → max possible blast radius
3. chaosServiceAccount                    → Kubernetes-level permission scope
4. appinfo.applabel + PODS_AFFECTED_PERC   → exact targets and percentage
5. TOTAL_CHAOS_DURATION + FORCE             → how long and how harsh
6. Probes (httpProbe/promProbe/etc.)          → automatic pass/fail gating
7. engineState: stop / delete ChaosEngine       → instant kill switch
```

---

This guide covers running LitmusChaos safely in production - scoped service accounts, blast radius control, RBAC layers, and the abort/kill-switch patterns every experiment needs, with visual representations of each safeguard.