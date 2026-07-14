# Chaos Mesh Experiment Types - PodChaos, NetworkChaos, StressChaos & More

The core fault-injection CRDs Chaos Mesh provides, with visual diagrams of each failure mode.

## Table of Contents
- [PodChaos](#podchaos)
- [NetworkChaos](#networkchaos)
- [StressChaos](#stresschaos)
- [IOChaos](#iochaos)
- [TimeChaos](#timechaos)
- [DNSChaos](#dnschaos)
- [HTTPChaos](#httpchaos)
- [KernelChaos](#kernelchaos)
- [Selecting Targets - mode and selector](#selecting-targets---mode-and-selector)

---

## PodChaos

**Simulates pod-level failures: killing a pod, killing a container inside a pod, or making a pod fail health checks.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-example
  namespace: my-app
spec:
  action: pod-failure     # pod-kill | pod-failure | container-kill
  mode: one
  duration: "30s"
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
```

**Visual:**
```
action: pod-kill
backend-1  Running → Terminating (immediately, permanently gone)
                      Kubernetes creates a replacement

action: pod-failure
backend-1  Running → NotReady for 30s → Running again
           (container paused/replaced with pause image, then restored)

action: container-kill
backend-1 [app, sidecar]
           app container killed, pod stays, kubelet restarts just that container
```

**Use cases:**
```
pod-kill      → test replica redundancy / failover speed
pod-failure   → test readiness probe behavior + traffic draining
container-kill→ test container restart policy & app startup resilience
```

---

## NetworkChaos

**Simulates degraded or broken networking - the most commonly used chaos type in real production testing.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-example
  namespace: my-app
spec:
  action: delay
  mode: all
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "25"
  duration: "2m"
```

**Visual:**
```
action: delay
frontend → backend    normal: 5ms
frontend → backend    with chaos: 200ms ± 50ms
Question: do timeouts/retries in the client handle this gracefully?

action: loss
frontend → backend    30% packet loss
Question: does the app retry, or throw errors to users?

action: partition
frontend  ✗✗✗ backend     ← full network split, zero connectivity
Question: does the system detect and fail over,
          or hang waiting on a dead connection?

action: bandwidth
frontend → backend    capped at 1mbit/s
Question: does a large payload/batch job degrade gracefully?

action: corrupt
frontend → backend    5% of packets corrupted
Question: does TCP retransmission or app-level checksums catch this?

action: duplicate
frontend → backend    5% of packets duplicated
Question: is the receiving service idempotent?
```

**All NetworkChaos actions at a glance:**
| Action | Simulates |
|---|---|
| `delay` | Added latency + jitter |
| `loss` | Packet loss percentage |
| `duplicate` | Duplicate packets |
| `corrupt` | Corrupted packet payloads |
| `bandwidth` | Bandwidth throttling |
| `partition` | Full network partition between selected pods |

---

## StressChaos

**Simulates CPU or memory pressure on a pod.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-example
  namespace: my-app
spec:
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
  stressors:
    cpu:
      workers: 2
      load: 80
    memory:
      workers: 1
      size: "512MB"
  duration: "5m"
```

**Visual:**
```
Before: backend-1  CPU 10%   Memory 100Mi
During: backend-1  CPU 90%   Memory 612Mi   ← stress injected

Questions to answer:
- Does HPA (Horizontal Pod Autoscaler) kick in and scale out?
- Does the readiness probe correctly mark the pod NotReady
  before it fully falls over (avoiding bad traffic routing)?
- Do OOM-prone pods get killed and rescheduled cleanly?
```

---

## IOChaos

**Injects disk I/O faults - latency, errors, or corrupted reads/writes - inside the pod's filesystem via a sidecar.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-delay-example
  namespace: my-app
spec:
  action: latency
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: database
  volumePath: /var/lib/data
  path: "**/*"
  delay: "100ms"
  percent: 50
  duration: "1m"
```

**Visual:**
```
App writes to /var/lib/data
Normal:  write() returns in <1ms
Chaos:   50% of write() calls delayed by 100ms

Questions to answer:
- Do write-heavy services (databases, queues) have timeouts
  tuned correctly for slow disks?
- Does a slow disk cascade into upstream request timeouts?
```

---

## TimeChaos

**Skews the system clock inside a pod's network namespace - useful for testing distributed systems and TLS cert handling.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: TimeChaos
metadata:
  name: clock-skew-example
  namespace: my-app
spec:
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
  timeOffset: "-1h"
  duration: "3m"
```

**Visual:**
```
Real time:    14:00:00
Pod sees:     13:00:00   ← skewed -1h

Questions to answer:
- Does JWT/token expiry logic break with clock drift?
- Do distributed locks or leader election (etcd/Consul) misbehave?
- Are logs/traces timestamped consistently across skewed pods?
```

---

## DNSChaos

**Injects DNS resolution failures or random/incorrect responses.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  name: dns-error-example
  namespace: my-app
spec:
  action: error         # error | random
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: frontend
  patterns:
    - "backend.my-app.svc.cluster.local"
  duration: "1m"
```

**Visual:**
```
action: error
frontend resolves "backend.my-app.svc.cluster.local" → NXDOMAIN

action: random
frontend resolves "backend.my-app.svc.cluster.local" → random garbage IP

Questions to answer:
- Does the app handle DNS failures with retries/backoff,
  or crash immediately on a resolution error?
```

---

## HTTPChaos

**Intercepts and manipulates HTTP requests/responses at the application layer - abort, delay, or modify content.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: http-abort-example
  namespace: my-app
spec:
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
  target: Request
  port: 8080
  path: "/api/orders"
  abort: true
  duration: "1m"
```

**Visual:**
```
target: Request, abort: true
Client → POST /api/orders → [connection aborted before reaching app]

target: Response, replace body
Backend → 200 OK {"orders": [...]} → intercepted → replaced with 500 error

Questions to answer:
- Does the frontend show a friendly error instead of crashing?
- Do downstream services correctly interpret a 5xx and retry
  or fall back to cached/default data?
```

---

## KernelChaos

**Injects kernel-level faults (e.g., syscall errors) via eBPF - the most invasive/advanced chaos type.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: KernelChaos
metadata:
  name: kernel-fault-example
  namespace: my-app
spec:
  mode: one
  selector:
    namespaces: ["my-app"]
    labelSelectors:
      app: backend
  failKernRequest:
    callchain:
      - funcname: "__x64_sys_mkdirat"
    failtype: 0
    probability: 50
```

**Visual:**
```
App calls mkdir() syscall
Normal:  succeeds
Chaos:   50% chance syscall returns an injected error

Use case: testing low-level error handling in storage-heavy
apps, rarely needed outside specialized infra teams.
```

---

## Selecting Targets - mode and selector

**Every experiment uses the same two building blocks to decide WHAT to break.**

```yaml
selector:
  namespaces: ["my-app"]
  labelSelectors:
    app: backend
mode: fixed-percent
value: "50"
```

**Visual:**
```
mode options:
one            → exactly 1 random matching pod
all            → every matching pod
fixed          → a fixed NUMBER of pods (value: "3")
fixed-percent  → a PERCENTAGE of matching pods (value: "50")
random-max-percent → up to a percentage, randomly chosen each run

Example with 10 matching pods, mode: fixed-percent, value: "30":
┌────────────────────────────────────────┐
│ pod1 pod2 pod3 pod4 pod5 ...  pod10    │
│  X    X    X                            │  ← 3 pods (30%) affected
└────────────────────────────────────────┘

Start small (mode: one) → build confidence → scale up (mode: all)
This is the standard progression for maturing a chaos practice.
```

---

## Visual Summary

```
Failure Domain     CRD            Typical Question Answered
────────────────────────────────────────────────────────────────
Compute/Pod         PodChaos       Does failover work?
Network              NetworkChaos   Do timeouts/retries work?
CPU/Memory           StressChaos    Does autoscaling/backpressure work?
Disk                 IOChaos        Are storage timeouts tuned right?
Clock                TimeChaos      Does token/lock logic survive skew?
DNS                  DNSChaos       Does the app handle resolution failure?
HTTP layer           HTTPChaos      Does the app handle bad responses?
Kernel/Syscall       KernelChaos    Does low-level error handling work?
```

---

This guide covers Chaos Mesh's core experiment types - the failure modes you can inject and the specific resilience questions each one answers, with visual representations of every fault.