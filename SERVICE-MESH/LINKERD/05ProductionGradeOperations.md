# Linkerd in Production - Upgrades, HA, Multicluster, Troubleshooting

Operational commands and patterns for running Linkerd reliably in real production Kubernetes clusters, with visual diagrams.

## Table of Contents
- [High Availability (HA) Install](#high-availability-ha-install)
- [Upgrading Linkerd](#upgrading-linkerd)
- [Multicluster Setup](#multicluster-setup)
- [Ingress Integration](#ingress-integration)
- [Resource Planning](#resource-planning)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [linkerd diagnostics](#linkerd-diagnostics)
- [Uninstalling Linkerd](#uninstalling-linkerd)
- [CI/CD Pipeline Pattern](#cicd-pipeline-pattern)

---

## High Availability (HA) Install

**Runs multiple replicas of every control plane component with anti-affinity and PodDisruptionBudgets.**

```bash
linkerd install --ha | kubectl apply -f -
```

**Visual:**
```
Standard Install                     HA Install
┌─────────────────┐                 ┌─────────────────┐
│ identity: 1 pod  │                 │ identity: 3 pods │
│ destination: 1   │                 │ destination: 3   │
│ proxy-injector: 1│                 │ proxy-injector: 3│
└─────────────────┘                 └─────────────────┘
Single point of failure             Spread across nodes/zones
                                     PodDisruptionBudget enforced
                                     Higher default resource requests

Use HA for: any production cluster
Use standard for: dev/test/demo clusters
```

---

## Upgrading Linkerd

### Check current vs latest version

```bash
linkerd version
linkerd upgrade --crds | kubectl apply -f -
linkerd upgrade | kubectl apply -f -
```

**Visual:**
```
Upgrade Order (important!):
1. Upgrade CLI first
   curl ... | sh    (or download matching version)

2. Upgrade CRDs
   linkerd upgrade --crds | kubectl apply -f -

3. Upgrade control plane
   linkerd upgrade | kubectl apply -f -

4. Upgrade viz extension
   linkerd viz install | kubectl apply -f -   (re-run with new version)

5. Restart data plane to pick up new proxy version
   kubectl rollout restart deploy -n my-app

6. Verify
   linkerd check
   linkerd check --proxy
```

**Version skew warning:**
```
Control Plane: 2.15.x         Data Plane Proxies: 2.13.x
                                        ↑
                          linkerd check --proxy will WARN
                          "data plane is out of date"

Rule: keep control plane and data plane proxy versions
      within Linkerd's supported skew window - don't let
      it drift for months.
```

---

## Multicluster Setup

**Connects two or more Kubernetes clusters so services can call each other transparently and securely across clusters.**

```bash
# On both clusters: must share the SAME trust anchor
linkerd multicluster install | kubectl apply -f -

# Link cluster "west" from cluster "east"
linkerd --context=west multicluster link --cluster-name west \
  | kubectl --context=east apply -f -
```

**Visual:**
```
Cluster: east                          Cluster: west
┌─────────────────────┐               ┌─────────────────────┐
│ frontend             │               │ backend              │
│                      │  ═══mTLS═══   │                      │
│ Service Mirror:      │  (via gateway) │ Service: backend     │
│ backend-west          │◄─────────────►│                      │
└─────────────────────┘               └─────────────────────┘

frontend calls "backend-west.my-app.svc.cluster.local"
as if it were local - traffic actually crosses clusters
through the linkerd-multicluster gateway, still mTLS-secured
```

### Check multicluster health

```bash
linkerd multicluster check
linkerd multicluster gateways
```

**Output Example:**
```
CLUSTER   ALIVE   NUM_SVC   LATENCY_P50
west      True    4         12ms
```

---

## Ingress Integration

**Linkerd works alongside your existing ingress controller (NGINX, Traefik, Contour, etc.) - the ingress pod itself gets meshed.**

```bash
kubectl annotate namespace ingress-nginx linkerd.io/inject=enabled
kubectl rollout restart deploy -n ingress-nginx
```

**Visual:**
```
Internet
   │
   ▼
┌─────────────────┐
│ Ingress Controller│  (meshed - has linkerd-proxy sidecar)
│ (nginx/traefik)   │
└────────┬─────────┘
         │ mTLS
         ▼
┌─────────────────┐
│ backend (meshed) │
└─────────────────┘

nginx-specific annotation needed for gRPC/h2c backends:
nginx.ingress.kubernetes.io/service-upstream: "true"
```

---

## Resource Planning

**Account for proxy overhead when sizing clusters and namespaces.**

**Visual:**
```
Per-Proxy Overhead (defaults):
┌───────────────────────────────┐
│ CPU request:     100m          │
│ CPU limit:        NONE (bursty)│
│ Memory request:  64Mi          │
│ Memory limit:    256Mi (HA)    │
└───────────────────────────────┘

Cluster of 500 pods:
500 × 100m CPU request  = 50 cores reserved just for proxies
500 × 64Mi memory       = 32Gi memory reserved just for proxies

Control plane (HA):
identity:        3 × (100m CPU, 100Mi mem)
destination:     3 × (100m CPU, 100Mi mem)
proxy-injector:  3 × (100m CPU, 100Mi mem)

Plan cluster autoscaler / node sizing to include this overhead
```

---

## Troubleshooting Checklist

```bash
# 1. Is the control plane healthy?
linkerd check

# 2. Are proxies healthy and up to date?
linkerd check --proxy

# 3. Is the specific pod meshed (2/2 not 1/1)?
kubectl get pod backend-xyz -n my-app

# 4. What does the proxy log say?
kubectl logs backend-xyz -c linkerd-proxy -n my-app

# 5. Is traffic actually reaching the proxy?
linkerd viz tap deploy/backend -n my-app

# 6. Is mTLS actually happening between the pods involved?
linkerd viz edges deploy -n my-app
```

**Visual:**
```
Common Issue → Root Cause → Fix
──────────────────────────────────────────────────────────
Pod stuck 1/1 not 2/2   → namespace not annotated,       → annotate ns +
                           or pod created before inject     rollout restart
                           was enabled

503s after mesh install → app doesn't handle SIGTERM      → add preStop hook,
                           gracefully / proxy shuts down     proxy has native-
                           before app finishes requests       sidecars support

High latency after mesh → default retry/timeout too       → tune ServiceProfile
                           aggressive, or proxy CPU          per-route, raise
                           throttled                          proxy-cpu-limit

"context deadline        → timeout in ServiceProfile is    → check downstream
 exceeded" errors          too short for the real p99          latency, adjust
                                                                timeout value

Cross-namespace calls    → AuthorizationPolicy denying     → check Server +
 return connection reset   the caller identity                AuthorizationPolicy
```

---

## linkerd diagnostics

**Deeper introspection commands for support/debugging.**

```bash
linkerd diagnostics proxy-metrics deploy/backend -n my-app
linkerd diagnostics endpoints backend.my-app.svc.cluster.local:8080
linkerd diagnostics controller-metrics
```

**Visual:**
```
diagnostics proxy-metrics  → raw Prometheus metrics straight from one proxy
diagnostics endpoints      → what the destination service actually resolves to
diagnostics controller-metrics → control plane's own internal health metrics

Use when linkerd viz aggregated views hide something -
go straight to the raw proxy/control-plane data.
```

---

## Uninstalling Linkerd

```bash
linkerd viz uninstall | kubectl delete -f -
linkerd multicluster uninstall | kubectl delete -f -
linkerd uninstall | kubectl delete -f -
```

**Visual:**
```
Uninstall order (reverse of install):
1. Uninstall extensions (viz, multicluster, jaeger)
2. Uninject data plane workloads first (optional but cleaner)
   kubectl get deploy -A -o yaml | linkerd uninject - | kubectl apply -f -
3. Uninstall control plane
4. CRDs removed last (or left in place if reinstalling later)

⚠️ Uninstalling control plane WITHOUT uninjecting workloads first
   leaves pods with dead proxies expecting a control plane -
   always uninject or delete/redeploy workloads.
```

---

## CI/CD Pipeline Pattern

**Typical production pipeline stage for mesh-aware deployments.**

**Visual:**
```
┌──────────────────────────────────────────────────────┐
│ Pipeline: deploy-backend                              │
├──────────────────────────────────────────────────────┤
│ 1. Build & push image                                  │
│ 2. helm upgrade backend ./chart                        │
│    (namespace already annotated linkerd.io/inject=enabled)│
│ 3. kubectl rollout status deploy/backend -n my-app      │
│ 4. linkerd check --proxy                                │
│ 5. linkerd viz stat deploy/backend -n my-app            │
│    → gate deployment on SUCCESS rate threshold          │
│ 6. (canary) apply/update HTTPRoute weights               │
│ 7. Flagger or manual promote to 100%                     │
└──────────────────────────────────────────────────────┘

Post-deploy automated gate example (pseudo):
SUCCESS_RATE=$(linkerd viz stat deploy/backend -n my-app -o json | jq '.[0].successRate')
if (( $(echo "$SUCCESS_RATE < 0.98" | bc -l) )); then
  echo "Rollback triggered"; kubectl rollout undo deploy/backend -n my-app
fi
```

---

## Visual Summary

```
Production Checklist:
☑ HA install (linkerd install --ha)
☑ Own trust anchor + cert-manager rotation
☑ default-inbound-policy: deny + explicit AuthorizationPolicy
☑ ServiceProfiles for critical services (retries/timeouts)
☑ Resource requests/limits tuned for proxy overhead
☑ Monitoring: linkerd viz + Grafana + long-term Prometheus
☑ Version skew alerting (control plane vs data plane)
☑ Canary releases via HTTPRoute/TrafficSplit + Flagger
☑ Runbook: linkerd check → check --proxy → viz tap → edges
```

---

This guide covers running Linkerd in production - HA, upgrades, multicluster, ingress, resource planning, and troubleshooting, with visual representations of each operational pattern.