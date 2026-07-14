# Chaos Mesh Production Safety - Blast Radius, RBAC, Safeguards

How to run chaos experiments in production without causing a real outage, with visual diagrams.

## Table of Contents
- [Why Safety Matters More Than the Fault Itself](#why-safety-matters-more-than-the-fault-itself)
- [Blast Radius Control](#blast-radius-control)
- [Namespace Scoping](#namespace-scoping)
- [Duration Limits and Auto-Recovery](#duration-limits-and-auto-recovery)
- [Pause and Abort](#pause-and-abort)
- [RBAC for Chaos Mesh](#rbac-for-chaos-mesh)
- [Progressive Rollout of Chaos Practice](#progressive-rollout-of-chaos-practice)
- [Pre-Flight Checklist](#pre-flight-checklist)

---

## Why Safety Matters More Than the Fault Itself

**The goal of chaos engineering is to find weaknesses in a controlled way - not to recreate a real incident. Every experiment needs an escape hatch.**

**Visual:**
```
Good Chaos Experiment                  Bad Chaos Experiment
┌──────────────────────┐              ┌──────────────────────┐
│ Small blast radius     │              │ Runs against ALL       │
│ Time-boxed             │              │ production pods        │
│ Auto-reverts           │              │ No duration limit       │
│ Monitored live         │              │ No one watching         │
│ Kill switch ready       │              │ No abort mechanism     │
└──────────────────────┘              └──────────────────────┘
     Controlled experiment                  Actual outage
```

---

## Blast Radius Control

**Always start with the smallest possible target and expand gradually.**

```yaml
spec:
  mode: fixed-percent
  value: "5"      # only 5% of matching pods, not "all"
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
      tier: canary          # target only the canary tier, not stable
```

**Visual:**
```
Progression over weeks/months:
Week 1:  mode: one              → 1 pod affected
Week 2:  mode: fixed-percent 5%  → small % affected
Week 3:  mode: fixed-percent 20% → wider blast radius
Month 2: mode: all (staging only, never jump straight to
                     "all" in production)

Rule: never increase blast radius AND experiment complexity
      (new fault type) at the same time - change one variable
```

---

## Namespace Scoping

**Chaos Mesh experiments are namespaced - use this to hard-limit what a given experiment CAN touch, regardless of label selector mistakes.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: safe-experiment
  namespace: my-app        # experiment lives here
spec:
  selector:
    namespaces: ["my-app"] # AND can only target this namespace
```

**Visual:**
```
Namespace: my-app                Namespace: payments-critical
┌─────────────────────┐         ┌─────────────────────┐
│ Chaos allowed here    │         │ Chaos NEVER allowed   │
│ (has RBAC + selector  │         │ here (no chaos-mesh   │
│  restricted to this   │         │ manager role granted, │
│  namespace)           │         │ enforced via RBAC)     │
└─────────────────────┘         └─────────────────────┘

Use Kubernetes RBAC + Chaos Mesh's own namespace scope
together - defense in depth against a mistyped selector.
```

---

## Duration Limits and Auto-Recovery

**Every experiment should have a `duration` - Chaos Mesh automatically reverts the fault when it expires, even if the controller pod restarts.**

```yaml
spec:
  duration: "60s"
```

**Visual:**
```
Without duration:
Experiment runs INDEFINITELY until manually deleted
⚠️ if the person running it forgets, or gets paged away
   mid-experiment, the fault stays injected forever

With duration: "60s":
0s ────────────────────────60s
│  fault injected          │ auto-reverted
└───────────────────────────┘
Guaranteed cleanup, no matter what happens to the operator
or even the chaos-controller-manager pod itself
```

---

## Pause and Abort

**Every running experiment can be paused or deleted immediately - the kill switch.**

```bash
# Pause (fault stays injected, but experiment stops progressing)
kubectl annotate podchaos safe-experiment -n my-app \
  experiment.chaos-mesh.org/pause=true

# Full abort - immediately removes the fault
kubectl delete podchaos safe-experiment -n my-app
```

**Visual:**
```
Emergency Abort Flow:
┌────────────────────────────────────────┐
│ On-call sees unexpected impact           │
│         │                                │
│         ▼                                │
│ kubectl delete podchaos <name> -n <ns>   │
│         │                                │
│         ▼                                │
│ Chaos Mesh immediately reverts the fault  │
│ (removes iptables rule / restores pod /   │
│  clears tc netem rule / etc.)             │
└────────────────────────────────────────┘

Always have this command ready BEFORE starting any
production experiment - literally paste it in the incident
channel at experiment start time.
```

---

## RBAC for Chaos Mesh

**Kubernetes RBAC controls who can create Chaos CRDs at all - this is your primary production guardrail.**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: chaos-mesh-manager
  namespace: my-app
rules:
  - apiGroups: ["chaos-mesh.org"]
    resources: ["podchaos", "networkchaos", "stresschaos"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
```

**Visual:**
```
Team Access Model:
┌────────────────────────────────────────────┐
│ Namespace: staging                           │
│  Role: chaos-mesh-manager                    │
│  Bound to: all-engineers group                │
│  → anyone can experiment freely               │
├────────────────────────────────────────────┤
│ Namespace: production                        │
│  Role: chaos-mesh-manager                    │
│  Bound to: sre-team group ONLY                │
│  → requires SRE approval / scheduled GameDay  │
└────────────────────────────────────────────┘
```

---

## Progressive Rollout of Chaos Practice

**A maturity model teams typically follow when adopting chaos engineering.**

**Visual:**
```
Level 1: Local/Dev
  Manually run single experiments, learn the CRDs

Level 2: Staging, Manual
  Team runs GameDay exercises in staging, on a schedule

Level 3: Staging, Automated
  Schedule CRD runs chaos nightly in staging automatically,
  gated on CI passing first

Level 4: Production, Manual, Small Blast Radius
  SRE-approved single experiments in production,
  mode: one, tight duration, live monitoring, abort ready

Level 5: Production, Automated, Continuous
  Scheduled low-blast-radius chaos runs continuously in prod
  (Netflix Chaos Monkey style), fully integrated with alerting
  and automatic abort-on-SLO-breach
```

---

## Pre-Flight Checklist

```
☐ Blast radius is the smallest useful size (mode: one or small %)
☐ Namespace selector is scoped correctly (double-checked, not "all")
☐ duration is set (no indefinite experiments)
☐ Dashboards open and being watched live (Grafana/Prometheus)
☐ Abort command ready and tested (kubectl delete <chaos-kind> <name>)
☐ Stakeholders/on-call notified of the window
☐ Rollback plan for the APPLICATION exists independent of chaos
  (e.g. if pod-kill reveals a real bug, can you roll back the deploy?)
☐ Experiment is version-controlled in Git, not ad-hoc kubectl apply
```

---

## Visual Summary

```
Safety Layers (defense in depth):
1. RBAC              → who can even create chaos CRDs
2. Namespace scope    → what namespace can be targeted
3. Label selector     → what specific pods within that namespace
4. mode/value          → how many/what percentage of those pods
5. duration            → how long the fault can possibly last
6. Live monitoring      → human eyes on dashboards during the run
7. Abort command        → instant kill switch, always ready
```

---

This guide covers running Chaos Mesh safely in production - blast radius control, RBAC, auto-recovery, and the abort/kill-switch patterns every experiment needs, with visual representations of each safeguard.