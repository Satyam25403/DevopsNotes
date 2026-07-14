# LitmusChaos in Production - CI/CD Integration, Upgrades, Troubleshooting

Operational patterns for running LitmusChaos reliably as part of a real DevOps pipeline, with visual diagrams.

## Table of Contents
- [Chaos in CI/CD Pipelines](#chaos-in-cicd-pipelines)
- [Chaos as a Deployment Gate](#chaos-as-a-deployment-gate)
- [litmusctl in Automation](#litmusctl-in-automation)
- [Upgrading LitmusChaos](#upgrading-litmuschaos)
- [Resource Planning](#resource-planning)
- [Multi-Cluster Fleet Management](#multi-cluster-fleet-management)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Uninstalling LitmusChaos](#uninstalling-litmuschaos)

---

## Chaos in CI/CD Pipelines

**Running a Litmus workflow as an automated pipeline stage, gated on probe results - resilience testing on every release, not just occasional GameDays.**

**Visual:**
```
┌──────────────────────────────────────────────────────┐
│ Pipeline: deploy-backend (staging)                       │
├──────────────────────────────────────────────────────┤
│ 1. Build & push image                                       │
│ 2. Deploy to staging                                         │
│ 3. litmusctl create workflow --workflow-manifest chaos.yaml   │
│ 4. Poll ChaosResult / probeSuccessPercentage                    │
│ 5. Assert: verdict == "Pass" for all steps                        │
│ 6. Promote to production if gate passed, else block + alert         │
└──────────────────────────────────────────────────────┘
```

### Example pipeline snippet

```bash
litmusctl create workflow \
  --project-id "$PROJECT_ID" \
  --cluster-id "$CLUSTER_ID" \
  --workflow-manifest chaos-workflow.yaml

sleep 90

VERDICT=$(kubectl get chaosresult backend-pod-delete-pod-delete -n my-app \
  -o jsonpath='{.status.experimentStatus.verdict}')

if [ "$VERDICT" != "Pass" ]; then
  echo "Resilience gate FAILED - blocking promotion"
  exit 1
fi
```

**Visual:**
```
Deploy → Run Workflow → Check ChaosResult → Gate
   │           │               │              │
   ▼           ▼               ▼              ▼
 staging   pod-delete +    verdict field    pass → promote
           httpProbe        (Pass/Fail)      fail → block +
                                                    alert team
```

---

## Chaos as a Deployment Gate

**Visual:**
```
Traditional Gates:               Chaos-Aware Gates:
┌──────────────────┐            ┌──────────────────┐
│ Unit tests pass      │            │ Unit tests pass      │
│ Integration tests     │            │ Integration tests     │
│ Security scan clean    │            │ Security scan clean    │
│ Manual approval          │            │ Litmus workflow verdict │
└──────────────────┘            │  == Pass                  │
                                  │ Manual approval              │
                                  └──────────────────┘

Litmus's built-in resilience score also gives you a
TREND metric to report over time, not just a pass/fail
gate per release - useful in retro/quarterly reviews.
```

---

## litmusctl in Automation

**The CLI is what makes Litmus scriptable in a pipeline instead of only clickable in ChaosCenter's UI.**

```bash
litmusctl config set-account \
  --endpoint=https://chaoscenter.my-org.com \
  --username=ci-bot \
  --password="$LITMUS_CI_PASSWORD"

litmusctl get projects
litmusctl get agents --project-id "$PROJECT_ID"
litmusctl create workflow --project-id "$PROJECT_ID" \
  --cluster-id "$CLUSTER_ID" \
  --workflow-manifest chaos-workflow.yaml
```

**Visual:**
```
CI Runner
┌───────────────────────┐
│ litmusctl (authenticated│
│  via CI service account) │
└───────────┬───────────┘
            │ GraphQL API calls
            ▼
┌───────────────────────┐
│ ChaosCenter server       │
└───────────┬───────────┘
            │ triggers
            ▼
┌───────────────────────┐
│ Target cluster's agent   │
│ runs the workflow          │
└───────────────────────┘
```

---

## Upgrading LitmusChaos

```bash
helm repo update
helm upgrade chaos litmuschaos/litmus -n litmus --version 3.x.x
```

**Visual:**
```
Upgrade Order:
1. Check release notes for CRD schema changes
   (ChaosEngine/ChaosExperiment fields do evolve between majors)
2. Stop/delete any actively running ChaosEngines before upgrading
   kubectl patch chaosengine <name> -n <ns> --type merge \
     --patch '{"spec":{"engineState":"stop"}}'
3. Upgrade ChaosCenter (control plane) first
   helm upgrade chaos litmuschaos/litmus -n litmus
4. Upgrade each connected Agent from the ChaosCenter UI
   (Environments → Agent → Upgrade)
5. Re-run one test workflow per cluster to confirm compatibility

⚠️ ChaosCenter and Agent versions can drift briefly during
   rollout, but don't leave a mismatch for long - check the
   compatibility matrix in release notes.
```

---

## Resource Planning

**Visual:**
```
ChaosCenter (once, control plane):
┌───────────────────────────────┐
│ litmusportal-server              │
│  CPU request:   250m              │
│  Memory request: 300Mi            │
│ litmusportal-frontend             │
│  CPU request:   100m              │
│  Memory request: 200Mi            │
│ mongodb                            │
│  CPU request:   500m              │
│  Memory request: 500Mi            │
│  (needs persistent storage!)       │
└───────────────────────────────┘

Per-Agent (per onboarded cluster):
┌───────────────────────────────┐
│ subscriber                        │
│  CPU request:   50m               │
│  Memory request: 100Mi            │
│ chaos-operator                     │
│  CPU request:   100m              │
│  Memory request: 100Mi            │
│ event-tracker                       │
│  CPU request:   50m                │
│  Memory request: 100Mi              │
└───────────────────────────────┘

Experiment pods themselves are short-lived Jobs - size them
based on the specific experiment (e.g. pod-cpu-hog needs
enough CPU limit to actually generate meaningful load).
```

---

## Multi-Cluster Fleet Management

**Unlike Chaos Mesh, ChaosCenter is explicitly designed to manage MANY clusters from one place - a real advantage for platform/SRE teams.**

**Visual:**
```
┌─────────────────────────────────────────────────┐
│                ChaosCenter (single pane)            │
├─────────────────────────────────────────────────┤
│ Project: platform-team                              │
│  Environment: staging-us     → Agent: cluster-a       │
│  Environment: staging-eu      → Agent: cluster-b       │
│  Environment: production-us    → Agent: cluster-c       │
│  Environment: production-eu     → Agent: cluster-d       │
└─────────────────────────────────────────────────┘

Run the SAME workflow definition against multiple
Environments to test consistency of resilience across
regions - a single dashboard shows resilience score
per-cluster side by side.
```

---

## Troubleshooting Checklist

```bash
# 1. Is the ChaosCenter control plane healthy?
kubectl get pods -n litmus

# 2. Is the target cluster's agent healthy?
kubectl get pods -n <agent-namespace>

# 3. Did the ChaosEngine actually start?
kubectl describe chaosengine <name> -n my-app

# 4. Check the experiment pod's logs directly
kubectl logs -n my-app -l name=pod-delete

# 5. Check the ChaosResult for probe details
kubectl get chaosresult <name> -n my-app -o yaml
```

**Visual:**
```
Common Issue → Root Cause → Fix
──────────────────────────────────────────────────────────
ChaosEngine stuck        → chaosServiceAccount lacks       → grant Role/
"Initialized"                RBAC permissions for the         RoleBinding per
                             target resource type               the experiment's
                                                                 default RBAC yaml

Agent shows "Disconnected"→ network/firewall blocking       → check outbound
in ChaosCenter                agent → ChaosCenter connection    connectivity from
                                                                 agent namespace

Probe always fails          → probe URL/query unreachable    → verify probe
                                from within the experiment       endpoint resolves
                                pod's network namespace           from inside cluster

ChaosResult shows            → FORCE: "true" causing hard      → set FORCE: "false"
unexpected failures             crash the app can't recover      or add graceful
                                 from gracefully                  shutdown handling

Workflow step never           → wrong template reference in     → validate Argo
starts                          Argo Workflow YAML                 Workflow YAML
                                                                    syntax (argo lint)
```

---

## Uninstalling LitmusChaos

```bash
kubectl delete chaosengine --all --all-namespaces
kubectl delete chaosexperiment --all --all-namespaces

helm uninstall chaos -n litmus
kubectl delete ns litmus

# on each onboarded cluster's agent namespace:
kubectl delete deploy subscriber chaos-operator event-tracker -n <agent-namespace>
```

**Visual:**
```
Uninstall order (important!):
1. Delete/stop all ChaosEngines first
   (prevents faults being left stuck injected)
2. Remove ChaosExperiment definitions per namespace
3. helm uninstall chaos -n litmus  (removes ChaosCenter)
4. Manually remove agent components from EACH onboarded
   cluster (ChaosCenter uninstall doesn't reach out to
   remote clusters automatically)
```

---

## Visual Summary

```
Production Checklist:
☑ Chaos workflows run as an automated CI/CD gate, not just manual GameDays
☑ litmusctl authenticated via a dedicated CI service account
☑ Every ChaosEngine has TOTAL_CHAOS_DURATION + at least one probe
☑ ChaosCenter Projects/Environments RBAC restricts prod triggering
☑ Resource overhead (ChaosCenter + per-cluster agents) planned for
☑ Multi-cluster resilience scores compared side-by-side for regional parity
☑ Upgrade ChaosCenter first, then agents, never mid-experiment
☑ Workflows version-controlled in Git (GitOps applied)
```

---

This guide covers running LitmusChaos operationally in production - CI/CD gating with litmusctl, upgrades, multi-cluster fleet management, and troubleshooting, with visual representations of each pattern.