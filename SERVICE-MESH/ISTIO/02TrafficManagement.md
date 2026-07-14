# Service Mesh (Istio) - Traffic Management

Controlling exactly how requests flow between services: routing rules, traffic splitting for canary releases, and resilience policies.

## Table of Contents
- [VirtualService: Routing Rules](#virtualservice-routing-rules)
- [DestinationRule: Subsets and Policies](#destinationrule-subsets-and-policies)
- [Traffic Splitting for Canary Releases](#traffic-splitting-for-canary-releases)
- [Request Routing by Header (A/B Testing)](#request-routing-by-header-ab-testing)
- [Retries and Timeouts](#retries-and-timeouts)
- [Circuit Breaking](#circuit-breaking)
- [Fault Injection for Resilience Testing](#fault-injection-for-resilience-testing)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## VirtualService: Routing Rules

**A VirtualService defines HOW requests to a service get routed — which version, under what conditions.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
```

**Visual:**
```
Without a VirtualService:
Request to "reviews" service → Kubernetes' default
   round-robin load balancing across ALL pods matching
   the service selector, regardless of version

With a VirtualService:
Request to "reviews" service → Istio checks the
   VirtualService rules → routes SPECIFICALLY to
   subset "v1" (defined in a DestinationRule) →
   fine-grained control over EXACTLY which version
   receives traffic, not just "any healthy pod"
```

---

## DestinationRule: Subsets and Policies

**A DestinationRule defines subsets (typically by version label) and connection policies for a destination.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

**Visual:**
```
Relationship between VirtualService and DestinationRule:
┌─────────────────────┐        ┌─────────────────────┐
│    VirtualService          │        │    DestinationRule           │
│  "WHERE should traffic       │  uses  │  "WHAT subsets exist,           │
│   for 'reviews' go?"           │───────→│   and what policies apply         │
│   → subset: v1                     │        │   to each?"                            │
└─────────────────────┘        └─────────────────────┘

VirtualService = the ROUTING decision
DestinationRule = the DEFINITION of what subsets
                  exist and how connections behave
```

---

## Traffic Splitting for Canary Releases

**Gradually shifting a percentage of traffic to a new version — the single most common real-world Istio use case.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v2
          weight: 10
```

**Visual:**
```
Traffic Split Visualization:
┌───────────────────────────────────┐
│         100 requests arrive              │
└─────────────┬─────────────────────┘
              │
      ┌───────┴───────┐
      ↓ 90%                  ↓ 10%
┌──────────┐        ┌──────────┐
│  reviews v1     │        │  reviews v2     │
│  (stable,          │        │  (new release,       │
│   proven)             │        │   being validated)         │
└──────────┘        └──────────┘

Gradual Rollout Progression:
v1: 100% / v2: 0%    (initial state)
v1: 90%  / v2: 10%    ← monitor error rates/latency
v1: 75%  / v2: 25%    ← still healthy? continue
v1: 50%  / v2: 50%    ← still healthy? continue
v1: 0%   / v2: 100%    ← full rollout complete

If v2 shows elevated errors at ANY stage, simply
revert the weights back to v1: 100% / v2: 0% —
INSTANTLY, without redeploying anything, since
this is just a routing rule change.
```

**Visual — why this beats a traditional deployment:**
```
Traditional rolling deployment:
New version gradually replaces old pods → if there's
a bug, SOME percentage of ALL users hit the broken
version, and rolling back means re-deploying the
OLD image again (slower)

Istio traffic splitting:
Both versions run simultaneously, full-scale, the
WHOLE time — only the ROUTING percentage changes.
Rolling back is instant (change a number), not a
redeployment.
```

---

## Request Routing by Header (A/B Testing)

**Route SPECIFIC users (not just a random percentage) to a specific version — useful for beta testers, internal QA, or A/B experiments.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            x-beta-user:
              exact: "true"
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
```

**Visual:**
```
Request WITH header "x-beta-user: true"
        ↓
Routed to reviews v2 (the new version)

Request WITHOUT that header (regular users)
        ↓
Routed to reviews v1 (the stable version)

This lets internal teams or opted-in beta users
test a new version in PRODUCTION, using real
production data/traffic patterns, while regular
users remain completely unaffected.
```

---

## Retries and Timeouts

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure
```

**Visual:**
```
Retry Flow:
Request sent → 5xx error received →
   Retry attempt 1 (up to 1s) → still fails →
   Retry attempt 2 (up to 1s) → still fails →
   Retry attempt 3 (up to 1s) → succeeds →
   Response returned to the caller
                (total elapsed: up to 3s,
                 bounded by the overall timeout)

Why perTryTimeout AND overall timeout both matter:
Without perTryTimeout, a single slow retry attempt
could consume the entire budget alone. Without an
overall timeout, retries could theoretically continue
indefinitely. Both bounds together keep worst-case
latency predictable.
```

⚠️ **Important caution:** `retryOn: 5xx` retries on ANY 5xx error — including from non-idempotent operations (like a payment charge). Retry policies should be reviewed per-service; blindly retrying a "charge credit card" call could cause a double charge if the original request actually succeeded but the response was lost.

---

## Circuit Breaking

**Stop sending traffic to an unhealthy service instance, preventing cascading failures across the mesh.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**Visual:**
```
Outlier Detection Flow:
Pod "reviews-v1-abc123" returns 5 consecutive 5xx errors
        ↓
Istio EJECTS this specific pod from the load balancing
pool for 30 seconds ("baseEjectionTime")
        ↓
Traffic is routed ONLY to the remaining healthy pods
of "reviews" during this ejection window
        ↓
After 30s, the ejected pod is given another chance —
if it's healthy again, it rejoins the pool; if it's
still failing, it gets ejected again (with an
increasing ejection time on repeated failures)

Why this matters:
A single misbehaving pod (out of many replicas) no
longer drags down the whole service's error rate —
traffic automatically avoids it while it recovers.
```

---

## Fault Injection for Resilience Testing

**Deliberately inject delays or errors to test how downstream services handle failure — without needing an actual outage.**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
      route:
        - destination:
            host: ratings
```

**Visual:**
```
What this does:
10% of requests to "ratings" get an artificial 5-second
delay injected — the OTHER 90% behave completely normally

Why this is valuable:
Does the CALLING service (e.g. "reviews", which depends
on "ratings") have a sensible timeout configured? Does
it show a reasonable fallback/degraded UI instead of
hanging indefinitely? Fault injection lets you TEST this
resilience behavior deliberately, in a controlled way,
without waiting for a real production incident to find out.
```

**Injecting errors instead of delays:**
```yaml
http:
  - fault:
      abort:
        percentage:
          value: 5
        httpStatus: 500
    route:
      - destination:
          host: ratings
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is rolling out a significant rewrite of the checkout service and wants to validate it against real production traffic gradually, with an instant rollback option if anything goes wrong.

**What they configure:**
1. Deploys the new checkout version (`v2`) alongside the existing stable version (`v1`), both running simultaneously at full scale — using a **DestinationRule** to define the two subsets by version label.
2. Starts with a **VirtualService weight of v1: 95% / v2: 5%**, routing only a small fraction of real production traffic to the new version while monitoring error rates and latency via Grafana (fed by Istio's automatic telemetry).
3. Additionally configures **header-based routing** so the internal QA team (identified via an `x-internal-qa: true` header set by their test harness) always hits v2 regardless of the percentage split — giving them dedicated, deterministic access to validate specific scenarios.
4. After 24 hours at 5% with no elevated errors, progressively increases to 25%, then 50%, then 100% — at each stage, watching the same dashboards before proceeding to the next stage.
5. Before the rollout, runs a **fault injection test** against a staging version of the OLD checkout service, injecting a 3-second delay on its call to the payment service, confirming the checkout flow correctly falls back to a "processing, please wait" UI state instead of hanging or erroring — validating this resilience behavior BEFORE it's ever needed for real.
6. When v2 briefly shows an elevated error rate at the 50% stage (traced to an edge case in a specific payment method), instantly reverts the VirtualService weight back to v1: 100% — a configuration change taking seconds, with zero redeployment needed.

**Why this matters:** The combination of gradual traffic splitting, header-based testing access, and proactive fault injection turns what could be a risky "flip the switch and hope" release into a controlled, observable, instantly-reversible rollout — the core promise of using a service mesh for deployment safety.

---

Next: **03security_mtls.md** — mutual TLS, PeerAuthentication, and AuthorizationPolicy for zero-trust service-to-service security.