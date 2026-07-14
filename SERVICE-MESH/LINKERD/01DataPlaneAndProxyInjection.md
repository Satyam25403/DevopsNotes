# Linkerd Data Plane & Proxy Injection - Visual Guide

How workloads get "meshed" - the linkerd-proxy sidecar, injection methods, and annotations used daily in DevOps pipelines.

## Table of Contents
- [What Injection Does](#what-injection-does)
- [Automatic Injection via Namespace Annotation](#automatic-injection-via-namespace-annotation)
- [Manual Injection via CLI](#manual-injection-via-cli)
- [Injection in CI/CD Pipelines](#injection-in-cicd-pipelines)
- [Per-Workload Injection Control](#per-workload-injection-control)
- [Proxy Resource Tuning](#proxy-resource-tuning)
- [Checking Meshed Status](#checking-meshed-status)
- [Uninjecting / Removing the Mesh](#uninjecting--removing-the-mesh)

---

## What Injection Does

**Injection adds two extra containers to every pod: `linkerd-proxy` and an init container (or CNI) that redirects traffic into it.**

**Visual:**
```
Before Injection:
Pod: frontend
┌─────────────────┐
│ Container:      │
│  app (1 of 1)   │
└─────────────────┘

After Injection:
Pod: frontend
┌───────────────────────────────┐
│ Init Container:               │
│  linkerd-init  (iptables rules)│  ← runs once, sets up redirection
├───────────────────────────────┤
│ Containers:                    │
│  app             (1 of 2)      │
│  linkerd-proxy    (2 of 2)      │  ← sidecar, stays running
└───────────────────────────────┘

Result: 2/2 Ready (not 1/1)
All inbound/outbound traffic now flows through linkerd-proxy
```

**Traffic redirection:**
```
Outbound from app:
app → (iptables redirect) → linkerd-proxy (outbound) → network → dest proxy

Inbound to app:
network → dest linkerd-proxy (inbound) → (iptables redirect) → app

App code is unaware - it still connects to "localhost" or service DNS names
```

---

## Automatic Injection via Namespace Annotation

**The most common production pattern - label the namespace, every new pod gets injected automatically.**

```bash
kubectl annotate namespace my-app linkerd.io/inject=enabled
```

**Visual:**
```
Before:
Namespace: my-app
┌─────────────────────┐
│ (no annotation)     │
└─────────────────────┘
New pods → NOT injected

After:
Namespace: my-app
┌─────────────────────────────────┐
│ linkerd.io/inject=enabled       │
└─────────────────────────────────┘
New pods → automatically injected
by the linkerd-proxy-injector webhook

⚠️ Existing pods are NOT retroactively injected
   You must roll them: kubectl rollout restart deploy -n my-app
```

### Rollout restart to inject existing workloads

```bash
kubectl rollout restart deployment -n my-app
```

**Visual:**
```
Deployment: backend (before)
Pod: backend-abc  1/1 Ready   ← not meshed

kubectl rollout restart triggers new ReplicaSet
Deployment: backend (after)
Pod: backend-xyz  2/2 Ready   ← meshed (webhook intercepted pod creation)
```

---

## Manual Injection via CLI

**Useful for one-off manifests, Helm charts without native support, or GitOps pipelines that pre-render YAML.**

```bash
kubectl get deploy backend -o yaml | linkerd inject - | kubectl apply -f -
```

**Visual:**
```
Plain manifest.yaml
        │
        ▼
  linkerd inject -   ← rewrites YAML, adds init container + proxy container
        │
        ▼
Injected manifest.yaml
        │
        ▼
  kubectl apply -f -
```

### Inject a manifest file directly (common in CI)

```bash
linkerd inject deployment.yaml | kubectl apply -f -
```

**Visual:**
```
CI Pipeline Stage:
┌────────────────────────────────────────────┐
│ 1. kustomize build . > manifest.yaml        │
│ 2. linkerd inject manifest.yaml > final.yaml│
│ 3. kubectl apply -f final.yaml              │
└────────────────────────────────────────────┘

Used when namespace-wide auto-inject is not enabled
or when workloads live across multiple namespaces
```

### Dry-run to preview the injected YAML

```bash
linkerd inject deployment.yaml --manual > injected.yaml
diff deployment.yaml injected.yaml
```

---

## Injection in CI/CD Pipelines

**Typical GitOps/Argo CD or Jenkins pattern for meshed deployments.**

**Visual:**
```
Git Repo (source of truth)
┌───────────────────────┐
│ deployment.yaml       │  (no linkerd config, plain manifest)
└───────────┬───────────┘
            │
            ▼
     CI Pipeline (Jenkins/GitHub Actions/GitLab CI)
┌──────────────────────────────────────────┐
│ Stage: Build → Test → Package             │
│ Stage: linkerd inject manifest.yaml       │
│ Stage: kubectl apply / helm upgrade       │
└──────────────────────────────────────────┘
            │
            ▼
     Kubernetes Cluster
┌───────────────────────┐
│ Pod: 2/2 Ready (meshed)│
└───────────────────────┘

Alternative (preferred in production):
Rely on namespace-level linkerd.io/inject=enabled
so CI doesn't need to run `linkerd inject` at all -
the proxy-injector webhook does it server-side.
```

---

## Per-Workload Injection Control

**Override the namespace default at the pod/deployment level using annotations in `spec.template.metadata.annotations`.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-batch-job
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled   # opt this workload OUT
```

**Visual:**
```
Namespace: my-app (inject=enabled)
┌───────────────────────────────────────────┐
│ Deployment: frontend   → injected (2/2)    │
│ Deployment: backend    → injected (2/2)    │
│ Deployment: batch-job  → skipped (1/1)     │  ← inject=disabled override
└───────────────────────────────────────────┘

Common reasons to skip:
- Short-lived batch/CronJobs where mesh adds no value
- Workloads that can't tolerate sidecar startup ordering
```

**Other useful pod annotations:**
```yaml
config.linkerd.io/proxy-cpu-limit: "500m"
config.linkerd.io/proxy-memory-limit: "128Mi"
config.linkerd.io/skip-outbound-ports: "5432,6379"   # skip DB/Redis ports
config.linkerd.io/skip-inbound-ports: "9090"          # skip metrics scrape port
```

---

## Proxy Resource Tuning

**Set default proxy CPU/memory at install time so every sidecar is right-sized for production.**

```bash
linkerd install \
  --proxy-cpu-request "100m" \
  --proxy-cpu-limit "500m" \
  --proxy-memory-request "64Mi" \
  --proxy-memory-limit "256Mi" \
  | kubectl apply -f -
```

**Visual:**
```
Per-Pod Overhead (typical):
┌─────────────────────────────┐
│ linkerd-proxy               │
│  CPU request:   100m         │
│  Memory request: 64Mi        │
└─────────────────────────────┘

Multiply by pod count in cluster:
100 pods × 64Mi  = 6.4Gi additional memory reserved
100 pods × 100m  = 10 cores additional CPU reserved

⚠️ Always account for this in cluster capacity planning
```

---

## Checking Meshed Status

### linkerd check --proxy

```bash
linkerd check --proxy
```

**Visual:**
```
Verifies:
✓ data plane proxies are up-to-date with control plane version
✓ proxies have valid mTLS certificates
✓ no proxies stuck in "not ready"
```

### Get meshed % per namespace

```bash
linkerd viz stat deploy -n my-app
```

**Output Example:**
```
NAME       MESHED   SUCCESS      RPS   LATENCY_P50   LATENCY_P99
frontend   3/3      100.00%    12.4rps      3ms          15ms
backend    2/2       99.85%     8.1rps      5ms          22ms
batch-job  0/1        -           -           -            -
```

---

## Uninjecting / Removing the Mesh

```bash
kubectl get deploy backend -o yaml | linkerd uninject - | kubectl apply -f -
```

**Visual:**
```
Injected Pod (2/2)               Uninjected Pod (1/1)
┌─────────────────┐             ┌─────────────────┐
│ app             │   uninject   │ app             │
│ linkerd-proxy   │  ────────→   │ (removed)       │
│ linkerd-init    │              │ (removed)       │
└─────────────────┘             └─────────────────┘

Use when: rolling back a bad mesh rollout,
migrating a workload out of the mesh, or debugging
whether an issue is proxy-related.
```

---

## Visual Summary

```
1. Enable at namespace level
   kubectl annotate ns my-app linkerd.io/inject=enabled

2. Roll existing workloads
   kubectl rollout restart deploy -n my-app

3. (Optional) Manual inject in CI
   linkerd inject manifest.yaml | kubectl apply -f -

4. Verify
   linkerd viz stat deploy -n my-app
   → MESHED column shows N/N

5. Tune resources
   config.linkerd.io/proxy-cpu-limit, proxy-memory-limit annotations

6. Exceptions
   linkerd.io/inject: disabled on specific workloads (batch jobs, DBs)
```

---

This guide covers how Linkerd's data plane proxy gets attached to workloads through automatic and manual injection, and how to control/tune it for real pipelines.