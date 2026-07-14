# LitmusChaos Workflows & Scheduling - Orchestrating Multi-Step Chaos

Chaining multiple experiments together with Argo Workflows and running chaos on a recurring schedule, with visual diagrams.

## Table of Contents
- [Litmus Workflows are Argo Workflows](#litmus-workflows-are-argo-workflows)
- [Building a Workflow via ChaosCenter](#building-a-workflow-via-chaoscenter)
- [Workflow YAML Structure](#workflow-yaml-structure)
- [Sequential vs Parallel Steps](#sequential-vs-parallel-steps)
- [Scheduling Recurring Chaos](#scheduling-recurring-chaos)
- [GitOps for Chaos Workflows](#gitops-for-chaos-workflows)
- [GameDay Pattern with Litmus](#gameday-pattern-with-litmus)

---

## Litmus Workflows are Argo Workflows

**Unlike a custom Workflow CRD, Litmus builds directly on top of Argo Workflows - meaning anything you know about Argo Workflows (DAGs, steps, templates) applies directly.**

**Visual:**
```
ChaosCenter "Chaos Workflow"
        │
        ▼
Underneath, this IS an Argo Workflow object
        │
        ▼
┌────────────────────────────────────┐
│ apiVersion: argoproj.io/v1alpha1     │
│ kind: Workflow                       │
│ spec:                                │
│   templates:                          │
│     - steps: [...]                     │
│       (each step = one ChaosEngine run) │
└────────────────────────────────────┘

Benefit: you can inspect/debug it with standard
Argo tooling (argo get, argo logs, Argo UI) in
addition to the ChaosCenter UI.
```

---

## Building a Workflow via ChaosCenter

**Visual, drag-and-drop workflow builder in the ChaosCenter UI - the most common way teams build multi-step scenarios.**

**Visual:**
```
ChaosCenter UI Flow:
┌────────────────────────────────────────────┐
│ 1. Chaos Workflows → Schedule a Workflow      │
│ 2. Choose target cluster (Agent)               │
│ 3. Add experiment(s) from ChaosHub              │
│    - pod-network-latency                        │
│    - pod-cpu-hog                                │
│    - pod-delete                                 │
│ 4. Arrange order (sequential or parallel)         │
│ 5. Add probes per step                            │
│ 6. Set schedule: run now / cron / one-time later  │
│ 7. Save & Run                                       │
└────────────────────────────────────────────┘

The UI generates the underlying Argo Workflow YAML for you -
you can export/inspect it and check it into Git afterward.
```

---

## Workflow YAML Structure

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: cascading-failure-test
  namespace: my-app
spec:
  entrypoint: custom-chaos
  templates:
    - name: custom-chaos
      steps:
        - - name: network-latency
            template: network-latency-chaos
        - - name: cpu-hog
            template: cpu-hog-chaos
        - - name: pod-delete
            template: pod-delete-chaos

    - name: network-latency-chaos
      inputs:
        artifacts:
          - name: network-latency-chaos
            path: /tmp/chaosengine-network-latency.yaml
            raw:
              data: |
                apiVersion: litmuschaos.io/v1alpha1
                kind: ChaosEngine
                metadata:
                  name: network-latency
                  namespace: my-app
                spec:
                  appinfo:
                    appns: my-app
                    applabel: "app=backend"
                    appkind: deployment
                  experiments:
                    - name: pod-network-latency
      container:
        image: litmuschaos/k8s:latest
        command: [sh, -c]
        args: ["kubectl apply -f /tmp/chaosengine-network-latency.yaml"]
```

**Visual:**
```
steps: [[A],[B],[C]]   → each inner list = parallel group
                          each outer list entry = sequential stage

- - name: A          ← Stage 1 (runs alone)
- - name: B          ← Stage 2 (runs after Stage 1 finishes)
- - name: C          ← Stage 3 (runs after Stage 2 finishes)
```

---

## Sequential vs Parallel Steps

**Visual:**
```
Sequential (default, separate stages):
steps:
  - - name: network-latency
  - - name: cpu-hog
  - - name: pod-delete

Timeline:
0min ──2min──4min──5min
│lat  │cpu   │kill │
└─────┴──────┴─────┘

Parallel (same stage, grouped in one inner list):
steps:
  - - name: network-latency
    - name: cpu-hog

Timeline:
0min ──────────2min
│ latency AND cpu │  ← both run simultaneously
└─────────────────┘
```

---

## Scheduling Recurring Chaos

**ChaosCenter lets you schedule a workflow to run once at a future time, or repeatedly via cron.**

**Visual:**
```
ChaosCenter → Schedule a Workflow → Reliability:
┌──────────────────────────────────────────┐
│ Run Type:                                   │
│  ○ Run once, now                             │
│  ○ Run once, at a specific time               │
│  ● Recurring (cron expression)                 │
│     "0 2 * * 0"   ← every Sunday at 2 AM        │
└──────────────────────────────────────────┘

Result: same as Chaos Mesh's Schedule CRD - moves chaos
from occasional manual GameDay to continuous automated
resilience testing.
```

### Via litmusctl

```bash
litmusctl create workflow \
  --project-id <id> \
  --workflow-manifest workflow.yaml \
  --cluster-id <agent-id>
```

---

## GitOps for Chaos Workflows

**Store workflow definitions in Git and let ChaosCenter/Argo sync them, rather than editing via the UI each time.**

**Visual:**
```
Git Repo: chaos-workflows/
┌───────────────────────────┐
│ cascading-failure-test.yaml│
│ nightly-pod-delete.yaml     │
│ network-latency-canary.yaml  │
└───────────────────────────┘
             │
             ▼ synced via ArgoCD/FluxCD, or applied by CI
┌───────────────────────────┐
│ Cluster: staging/production  │
│  Argo Workflow objects created│
└───────────────────────────┘

Benefit: chaos scenarios are versioned, reviewed via PR,
and reproducible across environments - same GitOps
discipline you already apply to app deployments.
```

---

## GameDay Pattern with Litmus

**Visual:**
```
GameDay Runbook (Litmus flavor):
┌──────────────────────────────────────────────────────┐
│ 1. Confirm resilience score baseline in ChaosCenter      │
│ 2. Run "cascading-failure-test" workflow live               │
│ 3. Watch probes (httpProbe/promProbe) in real time            │
│    via ChaosCenter's live workflow view                        │
│ 4. On-call responds as if it's a real incident                  │
│ 5. Review ChaosResult verdicts + resilience score delta           │
│ 6. Debrief, file tickets for any Fail verdicts                       │
└──────────────────────────────────────────────────────┘

ChaosCenter's built-in resilience score trend line makes
it easy to show leadership "our score went from 70% to
85% over the last quarter of GameDays."
```

---

## Visual Summary

```
1. Litmus Workflow      = an Argo Workflow under the hood
2. Build via ChaosCenter UI (drag/drop) or raw YAML
3. steps: [[..],[..]]    → sequential stages of parallel groups
4. Schedule                → one-time, future, or recurring cron
5. GitOps                   → version-control workflows, sync via ArgoCD/FluxCD
6. GameDay                    → same practice as Chaos Mesh, backed by
                                 resilience score trend in ChaosCenter
```

---

This guide covers orchestrating LitmusChaos experiments into multi-step scenarios using Argo-based Workflows and scheduling, with visual representations of each pattern.