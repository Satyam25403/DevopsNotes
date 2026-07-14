# Linkerd Observability - Golden Metrics, Dashboard, Tap, Grafana

Day-to-day observability commands DevOps engineers use to monitor meshed services in production, with visual diagrams.

## Table of Contents
- [Golden Metrics](#golden-metrics)
- [linkerd viz stat](#linkerd-viz-stat)
- [linkerd viz routes](#linkerd-viz-routes)
- [linkerd viz top](#linkerd-viz-top)
- [linkerd viz tap](#linkerd-viz-tap)
- [linkerd viz dashboard](#linkerd-viz-dashboard)
- [Grafana Integration](#grafana-integration)
- [Prometheus Integration](#prometheus-integration)
- [linkerd viz edges](#linkerd-viz-edges)

---

## Golden Metrics

**Linkerd automatically captures the "golden metrics" for every meshed HTTP/gRPC call, no app instrumentation needed.**

**Visual:**
```
┌────────────────────────────────────────────┐
│              GOLDEN METRICS                 │
├────────────────────────────────────────────┤
│ 1. Success Rate  → % of 2xx/OK responses    │
│ 2. RPS           → Requests per second      │
│ 3. Latency       → p50 / p95 / p99          │
└────────────────────────────────────────────┘

Collected automatically at the proxy level:
app ↔ linkerd-proxy ──metrics──→ Prometheus (scraped by linkerd-viz)
```

---

## linkerd viz stat

**Aggregate golden metrics for any Kubernetes resource type.**

```bash
linkerd viz stat deploy -n my-app
```

**Output Example:**
```
NAME       MESHED   SUCCESS      RPS   LATENCY_P50   LATENCY_P99
frontend   3/3      100.00%    12.4rps      3ms          15ms
backend    2/2       99.85%     8.1rps      5ms          22ms
```

### Stat by pod

```bash
linkerd viz stat pods -n my-app
```

### Stat namespace-wide

```bash
linkerd viz stat ns
```

**Visual:**
```
Resource types you can stat:
deploy, po, rs, ds, sts, job, svc, ns, ts (trafficsplit), au (authorization)

Time windows:
linkerd viz stat deploy -n my-app --time-window 5m
linkerd viz stat deploy -n my-app --time-window 1h
```

---

## linkerd viz routes

**Per-route metrics - requires a ServiceProfile to be applied.**

```bash
linkerd viz routes deploy/backend -n my-app
```

**Output Example:**
```
ROUTE               SERVICE   SUCCESS   RPS   LATENCY_P50   LATENCY_P99
GET /api/users      backend    99.9%   50rps       8ms          14ms
POST /api/orders     backend    98.2%   10rps      20ms          80ms
[DEFAULT]            backend   100.0%    2rps       4ms           9ms
```

**Visual:**
```
Without ServiceProfile:
linkerd viz routes deploy/backend
→ "no ServiceProfile found" - only aggregate stats available

With ServiceProfile applied:
→ per-endpoint breakdown, essential for pinpointing
  WHICH API route is degrading, not just the whole service
```

---

## linkerd viz top

**Live, real-time view of requests flowing through a resource (like `top` for HTTP traffic).**

```bash
linkerd viz top deploy/backend -n my-app
```

**Visual:**
```
Live updating table:
SOURCE          DESTINATION     METHOD  PATH             COUNT   SUCCESS  LATENCY
frontend-abc    backend-xyz     GET     /api/users        142     100%      8ms
frontend-abc    backend-xyz     POST    /api/orders        12      92%     45ms

Use case: real-time debugging during an incident,
watching which paths/pods are erroring RIGHT NOW
```

---

## linkerd viz tap

**Captures live request/response data (like tcpdump for HTTP), useful for deep debugging a single request flow.**

```bash
linkerd viz tap deploy/backend -n my-app
```

**Output Example:**
```
req id=1:1 proxy=in  src=10.1.2.3:54321 dst=10.1.4.5:8080
    :method=GET :authority=backend :path=/api/users
rsp id=1:1 proxy=in  src=10.1.2.3:54321 dst=10.1.4.5:8080
    :status=200 latency=8ms
end id=1:1 proxy=in  duration=8ms response-length=1240B
```

**Filter tap output:**
```bash
linkerd viz tap deploy/backend --path /api/orders --method POST -n my-app
linkerd viz tap deploy/backend --to deploy/database -n my-app
```

**Visual:**
```
tap = full packet capture of L7 traffic through proxies

Use sparingly in production:
- High volume services → tap generates a LOT of output
- Best for targeted debugging, not continuous monitoring
```

---

## linkerd viz dashboard

**Local web UI backed by the metrics-api and Prometheus.**

```bash
linkerd viz dashboard &
```

**Visual:**
```
┌───────────────────────────────────────────────┐
│  Linkerd Dashboard                             │
├───────────────────────────────────────────────┤
│ Namespaces │ Deployments │ Meshed │ Success   │
│ my-app     │ frontend    │ 3/3    │ 100%      │
│ my-app     │ backend     │ 2/2    │ 99.8%     │
├───────────────────────────────────────────────┤
│ [Topology Graph]                               │
│  frontend ──→ backend ──→ database             │
└───────────────────────────────────────────────┘

Runs on localhost via kubectl port-forward under the hood
Not exposed externally by default (security-first design)
```

---

## Grafana Integration

**Linkerd ships pre-built Grafana dashboards (Top Line, Deployment, Pod, Route, TCP dashboards).**

```bash
linkerd viz install --set grafana.enabled=true | kubectl apply -f -
```

**Visual:**
```
Prometheus (linkerd-viz)  ──scrapes──  linkerd-proxy /metrics
        │
        ▼
   Grafana Dashboards
┌───────────────────────────────┐
│ Linkerd Top Line               │
│ Linkerd Deployment              │
│ Linkerd Pod                     │
│ Linkerd Route                   │
│ Linkerd TCP                     │
└───────────────────────────────┘

Production tip: point Linkerd's Prometheus metrics to your
OWN long-term Prometheus/Thanos/Mimir stack instead of the
bundled short-retention one - bundled Prometheus is not
meant for long-term storage.
```

---

## Prometheus Integration

### Scraping proxy metrics directly (bypass linkerd-viz)

```bash
kubectl get --raw /api/v1/namespaces/my-app/pods/backend-xyz:4191/proxy/metrics
```

**Visual:**
```
Every linkerd-proxy exposes a /metrics endpoint on port 4191
┌─────────────────────────────────────┐
│ linkerd-proxy :4191/metrics          │
│  request_total                       │
│  response_latency_ms_bucket          │
│  tcp_open_connections                │
└─────────────────────────────────────┘

Your own Prometheus can scrape this directly via
ServiceMonitor/PodMonitor annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "4191"
```

---

## linkerd viz edges

**Shows the mTLS identity used between each pair of communicating pods - great for verifying zero-trust security posture.**

```bash
linkerd viz edges deploy -n my-app
```

**Output Example:**
```
SRC          DST        SRC_NS   DST_NS   SECURED
frontend     backend    my-app   my-app   √
backend      database   my-app   my-app   √
frontend     legacy-svc my-app   my-app   ×   ← NOT mTLS secured!
```

**Visual:**
```
√ = mTLS secured (both sides meshed, verified identity)
× = plaintext (one side not meshed - investigate/fix)

Used in security audits to prove/find gaps in mTLS coverage
```

---

## Visual Summary

```
Everyday observability toolkit:
linkerd viz stat      → aggregate golden metrics
linkerd viz routes    → per-route metrics (needs ServiceProfile)
linkerd viz top       → live traffic view
linkerd viz tap       → packet-level request capture
linkerd viz edges     → mTLS coverage between services
linkerd viz dashboard → web UI
Grafana + Prometheus  → long-term dashboards & alerting

Debugging flow during an incident:
1. linkerd viz stat deploy -n ns        → which service is unhealthy?
2. linkerd viz routes deploy/svc -n ns  → which route?
3. linkerd viz top deploy/svc -n ns     → live view, which pod/caller?
4. linkerd viz tap deploy/svc -n ns     → inspect actual request/response
```

---

This guide covers Linkerd's observability tooling - golden metrics, live traffic inspection, and dashboard/Grafana integration with visual representations of each command.