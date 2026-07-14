# Chaos Mesh in Production - CI/CD Integration, Upgrades, Troubleshooting

Operational patterns for running Chaos Mesh reliably as part of a real DevOps pipeline, with visual diagrams.

## Table of Contents
- [Chaos in CI/CD Pipelines](#chaos-in-cicd-pipelines)
- [Chaos as a Deployment Gate](#chaos-as-a-deployment-gate)
- [Upgrading Chaos Mesh](#upgrading-chaos-mesh)
- [Resource Planning](#resource-planning)
- [Multi-Cluster Considerations](#multi-cluster-considerations)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Comparison with Litmus](#comparison-with-litmus)
- [Uninstalling Chaos Mesh](#uninstalling-chaos-mesh)

---

## Chaos in CI/CD Pipelines

**Running chaos as an automated pipeline stage, not just a manual GameDay - moves resilience testing left, closer to every deploy.**

**Visual:**
```
┌──────────────────────────────────────────────────────┐
│ Pipeline: deploy-backend (staging)                     │
├──────────────────────────────────────────────────────┤
│ 1. Build & push image                                   │
│ 2. Deploy to staging                                     │
│ 3. Run smoke tests                                       │
│ 4. Apply chaos Workflow (network-delay + pod-kill)        │
│ 5. Run smoke tests AGAIN during chaos                     │
│ 6. Assert: success rate stayed > 95% during chaos          │
│ 7. Delete chaos experiment (or let duration expire)         │
│ 8. Promote to production if all steps passed                 │
└──────────────────────────────────────────────────────┘
```

### Example pipeline snippet

```bash
kubectl apply -f chaos/network-delay-experiment.yaml
sleep 60
SUCCESS_RATE=$(curl -s http://prometheus/api/v1/query \
  --data-urlencode 'query=sum(rate(http_requests_total{status="200"}[1m])) / sum(rate(http_requests_total[1m]))' \
  | jq '.data.result[0].value[1]')

kubectl delete -f chaos/network-delay-experiment.yaml

if (( $(echo "$SUCCESS_RATE < 0.95" | bc -l) )); then
  echo "Resilience gate FAILED - blocking promotion"
  exit 1
fi
```

**Visual:**
```
Deploy → Chaos Inject → Measure → Gate
   │           │            │        │
   ▼           ▼            ▼        ▼
 staging   pod-kill /   success   pass → promote
           net-delay    rate      fail → block +
                        check              alert team
```

---

## Chaos as a Deployment Gate

**Treat resilience the same way you treat test coverage or security scans - a required gate before promotion.**

**Visual:**
```
Traditional Gates:               Chaos-Aware Gates:
┌──────────────────┐            ┌──────────────────┐
│ Unit tests pass    │            │ Unit tests pass    │
│ Integration tests   │            │ Integration tests   │
│ Security scan clean │            │ Security scan clean │
│ Manual approval      │            │ Chaos experiment     │
└──────────────────┘            │  ran clean            │
                                  │ Manual approval        │
                                  └──────────────────┘

New question answered before every release:
"Does this version still survive a pod dying or
 network degrading, not just pass functional tests?"
```

---

## Upgrading Chaos Mesh

```bash
helm repo update
helm upgrade chaos-mesh chaos-mesh/chaos-mesh \
  -n chaos-mesh \
  --version 2.7.0
```

**Visual:**
```
Upgrade Order:
1. Check release notes for CRD changes
   (breaking CRD schema changes are common between minors)
2. Delete/pause any RUNNING experiments before upgrading
   kubectl delete podchaos,networkchaos --all -n my-app
3. helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh
4. kubectl get pods -n chaos-mesh   ← confirm all healthy
5. Re-apply/re-test one experiment to confirm compatibility

⚠️ Never upgrade Chaos Mesh WHILE an experiment is actively
   running - the controller-manager restart can leave a
   fault stuck injected with no active manager to revert it.
```

---

## Resource Planning

**Chaos Mesh components have modest but non-zero overhead - plan for it like any other cluster add-on.**

**Visual:**
```
Per-Node Overhead:
┌───────────────────────────────┐
│ chaos-daemon (DaemonSet)       │
│  CPU request:   50m             │
│  Memory request: 64Mi           │
│  Privileged (needs CAP_NET_ADMIN,│
│   CAP_SYS_ADMIN for tc/iptables/│
│   eBPF)                          │
└───────────────────────────────┘

Cluster-Wide (once):
┌───────────────────────────────┐
│ chaos-controller-manager        │
│  CPU request:   100m            │
│  Memory request: 128Mi          │
│ chaos-dashboard                 │
│  CPU request:   50m             │
│  Memory request: 64Mi           │
└───────────────────────────────┘

Security note: chaos-daemon requires privileged/host-level
capabilities - restrict which nodes/namespaces can schedule
it if running in a multi-tenant cluster.
```

---

## Multi-Cluster Considerations

**Chaos Mesh is installed per-cluster - there's no built-in multi-cluster orchestration (unlike Linkerd multicluster).**

**Visual:**
```
Cluster: staging               Cluster: production
┌─────────────────┐           ┌─────────────────┐
│ Chaos Mesh        │           │ Chaos Mesh        │
│ (separate install) │           │ (separate install, │
│                    │           │  stricter RBAC)     │
└─────────────────┘           └─────────────────┘

To run the SAME experiment across clusters:
- Store experiment YAML in Git (single source of truth)
- Apply via your existing GitOps tool (ArgoCD/FluxCD) per cluster
- Use different value overlays for blast radius per environment
  (staging: mode: all, production: mode: one)
```

---

## Troubleshooting Checklist

```bash
# 1. Are the core components healthy?
kubectl get pods -n chaos-mesh

# 2. Does the experiment show a status?
kubectl describe podchaos <name> -n my-app

# 3. Is chaos-daemon actually running on the target's node?
kubectl get pods -n chaos-mesh -o wide | grep chaos-daemon

# 4. Debug the actual injected fault (raw iptables/tc state)
chaosctl debug <podchaos-name> -n my-app

# 5. Check controller-manager logs for errors
kubectl logs -n chaos-mesh deploy/chaos-controller-manager
```

**Visual:**
```
Common Issue → Root Cause → Fix
──────────────────────────────────────────────────────────
Experiment stuck        → controller-manager crashed     → check controller
"Injecting" forever        mid-injection                    logs, delete/
                                                              recreate experiment

NetworkChaos has no      → chaosDaemon.runtime doesn't    → reinstall/upgrade
visible effect              match actual container           with correct
                             runtime (docker vs containerd)   runtime + socketPath

IOChaos has no effect     → volumePath doesn't match the   → confirm actual
                             pod's real mount path            mount path via
                                                              kubectl describe pod

Experiment fails RBAC     → ServiceAccount lacks chaos-    → grant Role/
error                        mesh.org apiGroup permissions    RoleBinding for
                                                              the CRD type

chaos-daemon CrashLoop    → insufficient privileged/host   → check node
                             capabilities on the node          security policies
                                                              (PSP/PSA) allow it
```

---

## Comparison with Litmus

**Both are CNCF chaos engineering projects for Kubernetes - worth knowing the difference when choosing or explaining a tool decision.**

**Visual:**
```
Chaos Mesh                          LitmusChaos
┌──────────────────────┐           ┌──────────────────────┐
│ CRD-first, declarative  │           │ ChaosHub - marketplace │
│ Strong dashboard + UI    │           │  of community experiments│
│ Fine-grained fault types │           │ ChaosCenter dashboard    │
│  (eBPF-based kernel      │           │ Strong GitOps + Argo      │
│   faults, IO faults)      │           │  Workflows integration     │
│ Simpler mental model      │           │ Broader experiment catalog│
└──────────────────────────┘           └──────────────────────┘

Choose Chaos Mesh when:  you want tight K8s-native CRDs and a
                          clean dashboard out of the box
Choose Litmus when:      you want a large pre-built experiment
                          catalog (ChaosHub) and deep Argo integration
```

---

## Uninstalling Chaos Mesh

```bash
kubectl delete podchaos,networkchaos,stresschaos,iochaos,timechaos,dnschaos,httpchaos,kernelchaos,workflow,schedule --all --all-namespaces

helm uninstall chaos-mesh -n chaos-mesh
kubectl delete ns chaos-mesh
```

**Visual:**
```
Uninstall order (important!):
1. Delete ALL running/scheduled experiments first
   (otherwise faults may be left stuck injected with
    no controller left to revert them)
2. helm uninstall chaos-mesh -n chaos-mesh
3. kubectl delete ns chaos-mesh
4. Verify no leftover iptables/tc rules on nodes
   (rare, but check via chaosctl or node access if
    something seems off post-uninstall)
```

---

## Visual Summary

```
Production Checklist:
☑ Chaos runs as an automated CI/CD gate, not just manual GameDays
☑ Every experiment has a duration + tested abort command
☑ RBAC restricts production chaos to SRE/approved roles
☑ Blast radius starts small and increases gradually over time
☑ Dashboards + Prometheus/Grafana watched live during every run
☑ Experiments version-controlled in Git (GitOps applied)
☑ Upgrade only when no experiments are actively running
☑ Resource overhead (privileged DaemonSet) accounted for in
  cluster capacity and security policy planning
```

---

This guide covers running Chaos Mesh operationally in production - CI/CD gating, upgrades, resource planning, and troubleshooting, with visual representations of each pattern.