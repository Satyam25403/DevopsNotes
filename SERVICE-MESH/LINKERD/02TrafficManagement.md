# Linkerd Traffic Management - Retries, Timeouts, Splits, Canary

Reliability and progressive delivery features Linkerd adds at the proxy layer, with visual diagrams.

## Table of Contents
- [Load Balancing (EWMA)](#load-balancing-ewma)
- [ServiceProfiles](#serviceprofiles)
- [Automatic Retries](#automatic-retries)
- [Timeouts](#timeouts)
- [Traffic Split (Canary Releases)](#traffic-split-canary-releases)
- [HTTPRoute (Gateway API Traffic Splitting)](#httproute-gateway-api-traffic-splitting)
- [Circuit Breaking](#circuit-breaking)
- [Traffic Shifting with Flagger](#traffic-shifting-with-flagger)

---

## Load Balancing (EWMA)

**Linkerd automatically load-balances requests across all endpoints of a Service using latency-aware EWMA (exponentially weighted moving average), not simple round-robin.**

**Visual:**
```
Without EWMA (round-robin):
Request 1 → Pod A (slow, 200ms)
Request 2 → Pod B (fast, 10ms)
Request 3 → Pod A (slow, 200ms)   ← keeps hitting the slow pod equally

With Linkerd EWMA:
Request 1 → Pod A (slow, 200ms)
Request 2 → Pod B (fast, 10ms)
Request 3 → Pod B (fast, 10ms)    ← proxy learns Pod A is slower, favors Pod B
Request 4 → Pod B (fast, 10ms)

No configuration needed - automatic per-request load balancing
```

---

## ServiceProfiles

**A ServiceProfile describes a service's routes so Linkerd can report per-route metrics and apply per-route retries/timeouts.**

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: backend.my-app.svc.cluster.local
  namespace: my-app
spec:
  routes:
  - name: GET /api/users
    condition:
      method: GET
      pathRegex: /api/users
    isRetryable: true
    timeout: 300ms
  - name: POST /api/orders
    condition:
      method: POST
      pathRegex: /api/orders
    timeout: 2s
```

**Generate one automatically from OpenAPI/protobuf:**
```bash
linkerd profile --open-api swagger.json backend > backend-profile.yaml
linkerd profile --proto backend.proto backend > backend-profile.yaml
kubectl apply -f backend-profile.yaml
```

**Visual:**
```
Without ServiceProfile:
linkerd viz stat deploy backend
→ shows aggregate metrics only (all routes mixed together)

With ServiceProfile:
linkerd viz routes deploy/backend
→ shows PER-ROUTE metrics:
  ROUTE               SUCCESS   RPS   LATENCY_P99
  GET /api/users       99.9%   50rps    12ms
  POST /api/orders     98.2%   10rps    80ms
  [DEFAULT]  (unmatched traffic falls here)
```

---

## Automatic Retries

**Retries are configured per-route in a ServiceProfile - Linkerd will not retry non-idempotent routes unless told they're safe.**

```yaml
spec:
  routes:
  - name: GET /api/users
    condition:
      method: GET
      pathRegex: /api/users
    isRetryable: true      # only retry safe (idempotent) routes
```

**Visual:**
```
Without Retries:
Client → proxy → backend Pod A (fails, 500) → Error returned to client

With Retries (isRetryable: true):
Client → proxy → backend Pod A (fails, 500)
                → proxy retries → backend Pod B (succeeds, 200)
                → Success returned to client

⚠️ Retry budget prevents retry storms:
retryBudget:
  retryRatio: 0.2        # max 20% of traffic can be retries
  minRetriesPerSecond: 10
```

**Retry budget visual:**
```
100 req/s normal traffic
Retry budget = 20% → max 20 additional retry req/s allowed
If failures spike beyond that → proxy stops retrying (protects backend)
```

---

## Timeouts

**Per-route timeout prevents slow requests from piling up.**

```yaml
spec:
  routes:
  - name: POST /api/orders
    timeout: 2s
```

**Visual:**
```
Without Timeout:
Client → proxy → backend (hangs indefinitely) → Client waits forever

With Timeout (2s):
Client → proxy → backend (hangs)
              → 2s elapsed → proxy cancels, returns 504 to client
              → Client fails fast instead of hanging
```

---

## Traffic Split (Canary Releases)

**Split traffic between two versions of a service by weight - classic canary deployment pattern (SMI TrafficSplit, being replaced by Gateway API HTTPRoute in newer Linkerd).**

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: backend-canary
  namespace: my-app
spec:
  service: backend         # the apex/root service clients call
  backends:
  - service: backend-v1
    weight: 900
  - service: backend-v2
    weight: 100
```

**Visual:**
```
Client requests → backend (apex service)
                       │
          ┌────────────┴────────────┐
          │ 90%                10% │
          ▼                        ▼
    backend-v1                backend-v2
    (stable)                  (canary)

Gradually shift weight over time:
Day 1: v1=900 v2=100   (10% canary)
Day 2: v1=500 v2=500   (50/50)
Day 3: v1=0   v2=1000  (100% canary - full rollout)

Rollback = just change weights back
```

### Apply and monitor a canary

```bash
kubectl apply -f backend-canary.yaml
linkerd viz stat ts/backend-canary -n my-app
```

**Output Example:**
```
NAME              APEX      LEAF          WEIGHT   SUCCESS   RPS
backend-canary    backend   backend-v1    900      99.9%    45.2rps
backend-canary    backend   backend-v2    100      97.1%     5.1rps
                                                     ↑
                                          lower success rate on canary
                                          → hold rollout / investigate
```

---

## HTTPRoute (Gateway API Traffic Splitting)

**Newer Linkerd versions use the Kubernetes Gateway API `HTTPRoute` for traffic splitting and routing (replacing SMI TrafficSplit).**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
  namespace: my-app
spec:
  parentRefs:
  - name: backend
    kind: Service
    group: core
    port: 80
  rules:
  - backendRefs:
    - name: backend-v1
      port: 80
      weight: 90
    - name: backend-v2
      port: 80
      weight: 10
```

**Visual:**
```
HTTPRoute (Gateway API)               TrafficSplit (SMI, legacy)
┌──────────────────────┐             ┌──────────────────────┐
│ Kubernetes-native      │            │ Linkerd/SMI-specific   │
│ standard API           │            │ being deprecated       │
│ path/header matching   │            │ weight-only splitting  │
│ + weighted splitting   │            │                       │
└──────────────────────┘             └──────────────────────┘

Recommendation: use HTTPRoute for new deployments
```

---

## Circuit Breaking

**Linkerd doesn't have a dedicated "circuit breaker" object - it achieves the same effect via a combination of retries, timeouts, and load-balancer endpoint ejection.**

**Visual:**
```
Failing Endpoint Detection:
┌────────────────────────────────────────┐
│ Pod A: 50% error rate over last N reqs │
│ Pod B: 0% error rate                   │
└────────────────────────────────────────┘
        │
        ▼
Load balancer (EWMA) sends fewer requests to Pod A automatically
        │
        ▼
Combined with timeouts + retries to Pod B
= effective circuit-breaking behavior without explicit config
```

---

## Traffic Shifting with Flagger

**In real production pipelines, Flagger automates canary promotion/rollback based on Linkerd metrics.**

**Visual:**
```
Flagger Control Loop:
┌─────────────────────────────────────────────────┐
│ 1. Deploy new version (backend-v2)               │
│ 2. Flagger creates TrafficSplit/HTTPRoute        │
│ 3. Shift 5% → wait → check linkerd metrics       │
│    success rate > 99%?  latency ok?              │
│ 4. Yes → shift to 10% → 20% → ... → 100%         │
│    No  → rollback to 0%, alert team              │
└─────────────────────────────────────────────────┘

deployment.yaml (Flagger Canary CRD)
┌───────────────────────────────┐
│ analysis:                     │
│   interval: 1m                │
│   threshold: 5                │
│   metrics:                    │
│   - name: request-success-rate│
│     thresholdRange:            │
│       min: 99                 │
│   - name: request-duration    │
│     thresholdRange:            │
│       max: 500                │
└───────────────────────────────┘
```

---

## Visual Summary

```
1. Define routes (optional but recommended)
   linkerd profile --open-api swagger.json svc > profile.yaml
   kubectl apply -f profile.yaml

2. Add resilience
   ServiceProfile: isRetryable + timeout per route

3. Progressive delivery
   TrafficSplit (SMI) or HTTPRoute (Gateway API) for weighted canary

4. Automate with Flagger
   Auto-promote/rollback based on linkerd viz metrics

5. Observe
   linkerd viz stat / linkerd viz routes / linkerd viz stat ts
```

---

This guide covers Linkerd's traffic management: load balancing, retries, timeouts, and canary/traffic-split releases with visual representations of each mechanism.