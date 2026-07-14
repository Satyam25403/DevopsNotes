# LitmusChaos Probes - Steady-State Validation

How Litmus verifies whether your application actually survived a chaos experiment, using built-in probes, with visual diagrams.

## Table of Contents
- [Why Probes Matter](#why-probes-matter)
- [HTTP Probe](#http-probe)
- [Command Probe](#command-probe)
- [Kubernetes Probe (k8sProbe)](#kubernetes-probe-k8sprobe)
- [Prometheus Probe (promProbe)](#prometheus-probe-promprobe)
- [Probe Modes](#probe-modes)
- [Combining Multiple Probes](#combining-multiple-probes)

---

## Why Probes Matter

**Without a probe, a ChaosEngine just injects a fault and reports "the pod was killed" - it has no idea if your APPLICATION actually stayed healthy. Probes are what turn a fault injection into an actual resilience test.**

**Visual:**
```
Without Probes:
ChaosEngine runs pod-delete → ChaosResult: verdict Pass
                                (Pass just means "pod-delete executed",
                                 NOT "your app handled it well")

With Probes:
ChaosEngine runs pod-delete
     │
     ▼
HTTP probe checks: GET /health returns 200 every 5s during chaos
     │
     ▼
ChaosResult: verdict Pass ONLY IF probe succeeded throughout
             (this now genuinely means "app stayed healthy")
```

---

## HTTP Probe

**Checks an HTTP endpoint continues to respond correctly during the experiment - the most commonly used probe.**

```yaml
spec:
  experiments:
    - name: pod-delete
      spec:
        probe:
          - name: "check-backend-health"
            type: "httpProbe"
            mode: "Continuous"
            runProperties:
              probeTimeout: 5
              interval: 2
              retry: 1
            httpProbe/inputs:
              url: "http://backend.my-app.svc.cluster.local/health"
              insecureSkipVerify: false
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
```

**Visual:**
```
Every 2s during the experiment:
GET http://backend.my-app.svc.cluster.local/health
Expect: HTTP 200

Timeline:
0s   2s   4s   6s   8s  ...
200  200  200  503  200   ← one blip
                ↑
        retry: 1 allows one failure before counting as a hard fail
```

---

## Command Probe

**Runs a shell command (inside a pod, or on the host via source) and checks its output/exit code - flexible for custom checks.**

```yaml
probe:
  - name: "check-queue-depth"
    type: "cmdProbe"
    mode: "Edge"
    runProperties:
      probeTimeout: 5
      interval: 5
      retry: 1
    cmdProbe/inputs:
      command: "kubectl get pods -n my-app -l app=backend --no-headers | wc -l"
      comparator:
        type: "int"
        criteria: ">="
        value: "2"
      source: "inline"
```

**Visual:**
```
Runs: kubectl get pods ... | wc -l
Expects: result >= 2

Use case: "at least 2 backend replicas must remain available
throughout the pod-delete experiment" - a structural check,
not just an HTTP health check.
```

---

## Kubernetes Probe (k8sProbe)

**Checks the state of a Kubernetes resource directly (e.g. a Deployment has the expected number of ready replicas).**

```yaml
probe:
  - name: "check-deployment-ready"
    type: "k8sProbe"
    mode: "Continuous"
    runProperties:
      probeTimeout: 5
      interval: 5
    k8sProbe/inputs:
      group: "apps"
      version: "v1"
      resource: "deployments"
      namespace: "my-app"
      resourceNames: "backend"
      operation: "present"
```

**Visual:**
```
Continuously checks:
Deployment/backend exists and matches "present" condition
in namespace my-app

Use case: confirm the Deployment object itself isn't
accidentally deleted/corrupted by an aggressive experiment
(e.g. node-drain affecting the whole namespace)
```

---

## Prometheus Probe (promProbe)

**Queries Prometheus directly and validates the result against a threshold - lets you gate chaos success on YOUR actual SLOs.**

```yaml
probe:
  - name: "check-error-rate"
    type: "promProbe"
    mode: "Continuous"
    runProperties:
      probeTimeout: 5
      interval: 10
      retry: 1
    promProbe/inputs:
      endpoint: "http://prometheus.monitoring.svc.cluster.local:9090"
      query: "sum(rate(http_requests_total{status=~\"5..\"}[1m])) / sum(rate(http_requests_total[1m]))"
      comparator:
        criteria: "<"
        value: "0.05"
```

**Visual:**
```
Every 10s during chaos:
Query Prometheus: error rate over last 1 minute
Expect: error rate < 5%

This is the STRONGEST probe type in production use -
it ties chaos pass/fail directly to your real SLO/error
budget, not a synthetic health check endpoint.
```

---

## Probe Modes

**Controls WHEN a probe runs relative to the fault injection.**

**Visual:**
```
SOT (Start of Test)   → runs once, right before chaos starts
                         (baseline / pre-check)

EOT (End of Test)      → runs once, right after chaos ends
                          (recovery confirmation)

Edge                     → runs once before AND once after
                           (SOT + EOT combined)

Continuous               → runs repeatedly throughout the ENTIRE
                            experiment duration
                            (most common for HTTP/Prometheus probes)

OnChaos                   → runs repeatedly, but ONLY during the
                            active fault-injection window
                            (excludes ramp-up/ramp-down time)
```

```
Timeline example (mode: OnChaos):
Ramp-up   [Chaos Injected Window]   Ramp-down
──────────┤▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓├──────────
           probe runs HERE only
```

---

## Combining Multiple Probes

**Real production ChaosEngines usually stack 2-3 probes for layered confidence.**

```yaml
probe:
  - name: "check-backend-health"
    type: "httpProbe"
    mode: "Continuous"
    # ...
  - name: "check-deployment-ready"
    type: "k8sProbe"
    mode: "Continuous"
    # ...
  - name: "check-error-rate"
    type: "promProbe"
    mode: "OnChaos"
    # ...
```

**Visual:**
```
Layered Validation:
┌────────────────────────────────────────┐
│ httpProbe   → is the endpoint reachable? │
│ k8sProbe     → is the Deployment intact?  │
│ promProbe     → is the real SLO holding?   │
└────────────────────────────────────────┘

ChaosResult verdict = Pass ONLY IF ALL probes pass
This is what makes a Litmus experiment a genuine
resilience TEST, not just a fault-injection script.
```

---

## Visual Summary

```
Probe Type    Checks                          Best For
──────────────────────────────────────────────────────────────
httpProbe      Endpoint returns expected code    App-level health
cmdProbe        Output of an arbitrary command     Custom/structural checks
k8sProbe         State of a K8s resource              Object-level integrity
promProbe         Prometheus query vs threshold        Real SLO/error-budget gating

Modes: SOT, EOT, Edge, Continuous, OnChaos
       → control WHEN the probe fires relative to the fault
```

---

This guide covers LitmusChaos probes - the mechanism that validates steady-state during and after chaos, turning fault injection into a genuine resilience test, with visual representations of each probe type and mode.